# SolutionRank244
Ryan Zhang  
Tuesday, May 05, 2015  

# 0 House Keeping
Set up working directory  设定工作环境    

```r
setwd("~/GitHub/edX/The Analytic Edge/Kaggle")
```

Load Libraries  装载函数包    

```r
library(tm)
library(rCUR)
library(topicmodels)
library(xgboost)
```

Function Definition   
Function for dummy encoding 用于生成虚拟编码的函数  

```r
dummyEncoding <- function(df, colname){
  dummyDf <- as.data.frame(model.matrix(~df[,colname]))
  names(dummyDf) <- paste(colname,as.character(levels(df[,colname])),sep="")
  df[,colname] <- NULL
  df <- cbind(df,dummyDf)
  df
  }
```

# 1 Data Preparing 数据准备工作  
Loading Data  

```r
newsTrain <- read.csv("NYTimesBlogTrain.csv", stringsAsFactors = F)
newsTest <- read.csv("NYTimesBlogTest.csv", stringsAsFactors = F)
ntrain <- nrow(newsTrain)
ntest <- nrow(newsTest)
Y <- as.factor(newsTrain$Popular)
newsTrain$Popular <- NULL
originalDf <- rbind(newsTrain, newsTest)
```

Imputing missing categorical data 填补下缺失数据  
I used a really simple approach, after inspecting the correlation among them.   
主要是通过SectionName来填补NewsDesk的值  

```r
imputedDf <- originalDf
iY <- Y
imputedDf$NewsDesk[originalDf$SectionName == "Arts"] <- "Culture"
imputedDf$NewsDesk[originalDf$SectionName == "Business Day"] <- "Business"
imputedDf$NewsDesk[originalDf$SectionName == "Crosswords/Games"] <- "Games"
imputedDf$NewsDesk[originalDf$SectionName == "Health"] <- "Science"
imputedDf$NewsDesk[originalDf$SectionName == "N.Y. / Region"] <- "Metro"
imputedDf$NewsDesk[originalDf$SectionName == "Open"] <- "OpEd"
imputedDf$NewsDesk[originalDf$SectionName == "Opinion"] <- "OpEd"
imputedDf$NewsDesk[originalDf$SectionName == "Technology"] <- "Tech"
imputedDf$NewsDesk[originalDf$SectionName == "Travel"] <- "Travel"
imputedDf$NewsDesk[originalDf$SectionName == "World"] <- "Foreign"
imputedDf$NewsDesk[originalDf$SubsectionName == "Education"] <- "Education"
```

Remove data entries which has a NewsDesk value not appeared in the testing data    
训练数据中有几个文章NewsDesk是National或者Sports，然而测试数据中没有这样的，干脆去掉  

```r
ntrain <- ntrain - sum(imputedDf$NewsDesk == "National")
iY <- iY[(imputedDf$NewsDesk[1:length(iY)] != "National")]
imputedDf <- imputedDf[(imputedDf$NewsDesk != "National"),]
ntrain <- ntrain - sum(imputedDf$NewsDesk == "Sports")
iY <- iY[(imputedDf$NewsDesk[1:length(iY)] != "Sports")]
imputedDf <- imputedDf[(imputedDf$NewsDesk != "Sports"),]
```

Change "" to "Other", No effect on modeling, just don't like to see ""   
将""改成"Other"，应该没什么影响

```r
for (i in 1:nrow(imputedDf)){
  for (j in 1:3){
    if (imputedDf[i,j] == ""){
      imputedDf[i,j] <- "Other"}}}
```

Log Transform "WordCount" 将字数做个对数转变     

```r
imputedDf$WordCount <- log(imputedDf$WordCount + 1)
```

# 2 Feature Extraction 提取特征
QR = Question Mark in the title?    
QR表示标题中有无问号    

```r
imputedDf$QR <- rep(0,nrow(imputedDf))
imputedDf$QR[grep("\\?", imputedDf$Headline)] <- 1
```

Extract Hour and day in week   
从发表日期中抽取时间和周几  

```r
imputedDf$PubDate <- strptime(imputedDf$PubDate, "%Y-%m-%d %H:%M:%S")
imputedDf$Hour <- as.factor(imputedDf$PubDate$h)
imputedDf$Wday <- as.factor(imputedDf$PubDate$wday)
imputedDf$PubDate <- NULL
```

Extract all headline and abstract to form a corpus    
抽取题名和摘要文本构建一个语料库    

```r
text <- vector()
for (i in 1:nrow(imputedDf)) {
  text <- rbind(text, paste(imputedDf$Headline[i], " ", imputedDf$Abstract[i]))
}
Corpus <- Corpus(VectorSource(text))
```

Corpus processing     
语料库处理     

```r
Corpus <- tm_map(Corpus, tolower)     
Corpus <- tm_map(Corpus, PlainTextDocument)    
Corpus <- tm_map(Corpus, removePunctuation)    
Corpus <- tm_map(Corpus, removeWords, stopwords("english"))     
Corpus <- tm_map(Corpus, stemDocument)
Corpus <- tm_map(Corpus, removeWords, c("york","time","today","day","said","say","report","week","will","year","articl","can","daili","news"))
```

Document ~ TF-IDF matrix And Document ~ TF matrix    
构建文档~TF-IDF矩阵 以及文档~TF矩阵     

```r
dtm <- DocumentTermMatrix(Corpus, control = list(weighting = weightTfIdf))   
tfdtm <- DocumentTermMatrix(Corpus)   
```

Get frequent terms matrix as feature   
抽取频繁的术语    

```r
sparse <- removeSparseTerms(dtm, 0.975)
words <- as.data.frame(as.matrix(sparse))
terms <- names(words)
row.names(words) <- c(1:nrow(imputedDf))
imputedDf <- cbind(imputedDf, words)
```


```r
kmclusters <- kmeans(words, 8, iter.max = 10000)
imputedDf <- cbind(imputedDf, kmclusters$cluster)
```

PCA 主要成分分析

```r
cMatrix <- cMatrix <- as.matrix(removeSparseTerms(dtm, 0.99))
s <- svd(cMatrix)
Sig <- diag(s$d)
# plot(s$d)
totaleng <- sum(Sig^2)
engsum <- 0
k <- 0

for (i in 1:nrow(Sig)){
  engsum <- engsum + Sig[i,i]^2
  if (engsum/totaleng > 0.8){
    k <- i
    break}}

set.seed(1126)
res <- CUR(cMatrix, c = 9, r = 85, k = k)
ncolC <- ncol(getC(res))
Ak <- getC(res) %*% getU(res)[,1:ncolC]
AkDF <- as.data.frame(Ak)
imputedDf <- cbind(imputedDf, AkDF)
```

LDA

```r
lda <- LDA(tfdtm, control = list(seed = 880306, alpha = 0.1), k = 3)
imputedDf$topic3 <- topics(lda)
lda <- LDA(tfdtm, control = list(seed = 880306, alpha = 0.1), k = 5)
imputedDf$topic5 <- topics(lda)
lda <- LDA(tfdtm, control = list(seed = 880306, alpha = 0.1), k = 7)
imputedDf$topic7 <- topics(lda)
lda <- LDA(tfdtm, control = list(seed = 880306, alpha = 0.1), k = 9)
imputedDf$topic9 <- topics(lda)
```

Dummy Encoding

```r
trainingDf <- imputedDf[,c(1:3,7,9:ncol(imputedDf))]
names(trainingDf) <- c(names(trainingDf)[1:52],"cluster",names(trainingDf)[54:ncol(trainingDf)])
trainingDf$SectionName <- as.factor(trainingDf$SectionName)
trainingDf$SubsectionName <- as.factor(trainingDf$SubsectionName)
trainingDf$NewsDesk <- as.factor(trainingDf$NewsDesk)
trainingDf$Hour <- as.factor(trainingDf$Hour)
trainingDf$Wday <- as.factor(trainingDf$Wday)
trainingDf$cluster <- as.factor(trainingDf$cluster)
trainingDf$topic3 <- as.factor(trainingDf$topic3)
trainingDf$topic5 <- as.factor(trainingDf$topic5)
trainingDf$topic7 <- as.factor(trainingDf$topic7)
trainingDf$topic9 <- as.factor(trainingDf$topic9)
toEnco <- c("NewsDesk","SectionName","SubsectionName","Hour","Wday","cluster","topic3","topic5","topic7","topic9")
for (colname in toEnco){
  trainingDf <- dummyEncoding(trainingDf, colname)
}
x <- trainingDf[1:ntrain,]
xt <- trainingDf[((1+ntrain):nrow(trainingDf)),]
```

# 3 Model Fitting 模型拟合    
Using cross validation to pick number of rounds 

```r
dtrain <- xgb.DMatrix(data = as.matrix(x), label = as.numeric(as.character(iY)))
nround <- 30
param <- list(max.depth=6, eta=0.5,silent=1,nthread = 4, objective='binary:logistic')
set.seed(880306)
cvresult <- xgb.cv(param, dtrain, nround, nfold=5, metrics={'auc'},showsd = FALSE)
nround <- which.max(cvresult$test.auc.mean)
```


```r
set.seed(880306)
dtrain <- xgb.DMatrix(data = as.matrix(x), label = as.numeric(as.character(iY)))
bst <- xgboost(data = dtrain,
               max.depth = 6,
               eta = 0.5,
               nround = nround,
               nthread = 4,
               eval_metric = "auc",
              objective = "binary:logistic")
```


```r
XgboostPred <- predict(bst, newdata = as.matrix(xt))
MySubmission = data.frame(UniqueID = newsTest$UniqueID, Probability1 = XgboostPred)
write.csv(MySubmission, "Xgboost880306.csv", row.names = F)
```
