# LAB1
Detect Pneumonia from chest X-ray images
import numpy as np
import torch
import torch.nn as nn
from torch.utils.data import DataLoader
import torchvision
from torchvision import datasets
import torchvision.transforms as transforms
import os
import matplotlib.pyplot as plt
from torchvision import models
import time
from tqdm import tqdm
import warnings
import copy
warnings.simplefilter("ignore")
warnings.filterwarnings("ignore", category=DeprecationWarning)
warnings.filterwarnings("ignore", category=UserWarning)
warnings.filterwarnings("ignore", category=FutureWarning)
from torchsummary import summary
from sklearn.metrics import accuracy_score,classification_report, f1_score,roc_auc_score, confusion_matrix, ConfusionMatrixDisplay
import seaborn as sns 

def images_transforms(phase):
    if phase == 'training':
        data_transformation =transforms.Compose([
            transforms.Resize(IMAGE_SIZE),
            transforms.RandomEqualize(10),
            transforms.RandomRotation(degrees=(-15,15)),
            #transforms.CenterCrop(64),
            transforms.ToTensor(),
            transforms.Normalize([0.485, 0.456, 0.406],[0.229, 0.224, 0.225])
        ])
    else:
        data_transformation=transforms.Compose([
            transforms.Resize(IMAGE_SIZE),
            transforms.ToTensor(),
            transforms.Normalize([0.485, 0.456, 0.406],[0.229, 0.224, 0.225])
        ])


    return data_transformation

class ResNet152(nn.Module):
   def __init__(self,num_class,pretrained_option=True):
        super(ResNet152,self).__init__()
        self.model=models.resnet152(pretrained=pretrained_option)

        #for param in self.model.parameters():
        #    param.requires_grad=False

        num_neurons=self.model.fc.in_features
        self.model.fc=nn.Linear(num_neurons,num_class)

   def forward(self,X):
        out=self.model(X)
        return out

class ResNet50(nn.Module):
   def __init__(self,num_class,pretrained_option=True):
        super(ResNet50,self).__init__()
        self.model=models.resnet50(pretrained=pretrained_option)

        if pretrained_option==True:
            for param in self.model.parameters():
                param.requires_grad=False

        num_neurons=self.model.fc.in_features
        self.model.fc=nn.Linear(num_neurons,num_class)

   def forward(self,X):
        out=self.model(X)
        return out

def training(model, train_loader, test_loader, Loss, optimizer, epochs, device, num_class, name):
    model.to(device)
    best_model_wts = None
    best_evaluated_acc = 0
    train_acc = []
    test_acc = []
    test_Recall = []
    test_Precision = []
    test_F1_score = []
    epoch_count = []
    scheduler = torch.optim.lr_scheduler.ExponentialLR(optimizer , gamma = 0.96)
    for epoch in range(1, epochs+1):
        with torch.set_grad_enabled(True):
            model.train()
            total_loss=0
            correct=0
            for idx,(data, label) in enumerate(tqdm(train_loader)):
                optimizer.zero_grad()

                data = data.to(device,dtype=torch.float)
                label = label.to(device,dtype=torch.long)

                predict = model(data)

                loss = Loss(predict, label.squeeze())

                total_loss += loss.item()
                pred = torch.max(predict,1).indices
                correct += pred.eq(label).cpu().sum().item()

                loss.backward()
                optimizer.step()

            total_loss /= len(train_loader.dataset)
            correct = (correct/len(train_loader.dataset))*100.
            print ("Epoch : " , epoch)
            print ("Loss : " , total_loss)
            print ("Correct : " , correct)

        scheduler.step()
        accuracy  , Recall , Precision , F1_score = evaluate(model, device, test_loader,epoch)
        train_acc.append(correct)
        test_acc.append(accuracy)
        test_Recall.append(Recall)
        test_Precision.append(Precision)
        test_F1_score.append(F1_score)
        epoch_count.append(epoch)



        if accuracy > best_evaluated_acc:
            best_evaluated_acc = accuracy
            best_model_wts = copy.deepcopy(model.state_dict())
    plt.clf()
    plt.title("Train Accuracy")
    plt.plot(epoch_count, train_acc)
    plt.show()
    plt.savefig("Train_Accuracy.png")

    plt.clf()
    plt.title("Test Accuracy")
    plt.plot(epoch_count, test_acc,'r')
    plt.show()
    plt.savefig("Test_Accuracy.png")

    plt.clf()
    plt.title("F1 Score")
    plt.plot(epoch_count, test_F1_score,'g')
    plt.show()
    plt.savefig("F1_Score.png")
    #save model
    torch.save(best_model_wts, name+".pt")
    model.load_state_dict(best_model_wts)

    return train_acc , test_acc , test_Recall , test_Precision , test_F1_score

def evaluate(model, device, test_loader,epoch):
    correct=0
    TP=0
    TN=0
    FP=0
    FN=0
    y_pred = []
    y_label = []
    with torch.set_grad_enabled(False):
        model.eval()
        for idx,(data,label) in enumerate(test_loader):
            data = data.to(device,dtype=torch.float)
            label = label.to(device,dtype=torch.long)
            predict = model(data)
            pred = torch.max(predict,1).indices
            #correct += pred.eq(label).cpu().sum().item()

            for j in range(data.size()[0]):
                #print ("{} pred label: {} ,true label:{}" .format(len(pred),pred[j],int(label[j])))
                y_pred.append(int (pred[j]))
                y_label.append(int (label[j]))
                if (int (pred[j]) == int (label[j])):
                    correct +=1
                if (int (pred[j]) == 1 and int (label[j]) ==  1):
                    TP += 1
                if (int (pred[j]) == 0 and int (label[j]) ==  0):
                    TN += 1
                if (int (pred[j]) == 1 and int (label[j]) ==  0):
                    FP += 1
                if (int (pred[j]) == 0 and int (label[j]) ==  1):
                    FN += 1

        matrix = confusion_matrix(y_label,y_pred)
        disp = ConfusionMatrixDisplay(confusion_matrix=matrix,display_labels=["Normal","Pneumonia"])
        disp.plot()
        plt.xlabel("Predicted")
        plt.ylabel("True")
        plt.show()
        save = ("confusion_",str(epoch),".png")
        plt.savefig("".join(save))
        
        print ("TP : " , TP)
        print ("TN : " , TN)
        print ("FP : " , FP)
        print ("FN : " , FN)

        print ("num_correct :",correct ," / " , len(test_loader.dataset))
        Recall = TP/(TP+FN)
        print ("Recall : " ,  Recall )

        Precision = TP/(TP+FP)
        print ("Preecision : " ,  Precision )

        F1_score = 2 * Precision * Recall / (Precision + Recall)
        print ("F1 - score : " , F1_score)

        correct = (correct/len(test_loader.dataset))*100.
        print ("Accuracy : " , correct ,"%")

    return correct , Recall , Precision , F1_score

if __name__=="__main__":
    IMAGE_SIZE=(128,128)
    batch_size=128
    learning_rate = 0.0005
    epochs=10
    num_classes=2

    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    print (device)

    train_path='archive/chest_xray/train'
    test_path='archive/chest_xray/test'
    val_path='archive/chest_xray/val'

    trainset=datasets.ImageFolder(train_path,transform=images_transforms('train'))
    testset=datasets.ImageFolder(test_path,transform=images_transforms('test'))
    valset=datasets.ImageFolder(val_path,transform=images_transforms('val'))

    train_loader = DataLoader(trainset,batch_size=batch_size,shuffle=True)
    test_loader = DataLoader(testset,batch_size=batch_size,shuffle=False)
    val_loader = DataLoader(valset,batch_size=batch_size,shuffle=True)

    model = ResNet152(2, True)
    #model = ResNet50(2, True)
    criterion = nn.CrossEntropyLoss()
    #optimizer = torch.optim.SGD(model.parameters(), lr=learning_rate,  momentum=0.9)
    optimizer = torch.optim.Adam(model.parameters(), lr=learning_rate)
    Loss = nn.CrossEntropyLoss()

    train_acc , test_acc , test_Recall , test_Precision , test_F1_score  = training(model, train_loader, test_loader, Loss, optimizer,epochs, device, num_classes, 'CNN_chest')
