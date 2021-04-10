# rstats-gsoc2021

#### Author: Jiarui      
#### Date: 2021-April-11th

### Q1:
Begin by downloading and building the ExpectedReturns and FactorAnalytics packages locally. Work through, and list any build errors or issues you encounter on install

* ##### Error:
```
ERROR: dependency ‘FactorAnalytics’ is not available for package ‘ExpectedReturns’
```
##### Solution:
Change the sequence and install FactorAnalytics first


* ##### Error:
FactorAnalysis depends on R with newer version. Mine is 3.5.2 
##### Solution:
Go to https://uvastatlab.github.io/phdplus/installR.html:
```
install.packages("installr")
library(installr)
updateR()
```


* ##### Error:
```
configure: error: GNU MP not found, or not 4.1.4 or up, see http://gmplib.org
ERROR: configuration failed for package ‘gmp’
removing ‘/home/ubuntu/R/x86_64-pc-linux-gnu-library/4.0/gmp’
ERROR: dependency ‘gmp’ is not available for package ‘Rmpfr’
Header file mpfr.h not found; maybe use --with-mpfr-include=INCLUDE_PATH
ERROR: configuration failed for package ‘Rmpfr’
```
##### Solution:
I go to http://cran.r-project.org/bin/macosx/RMacOSX-FAQ.html#Internationalization-of-the-R_002eapp and set its default to en_US.UTF-8
```
defaults write org.R-project.R force.LANG en_US.UTF-8
```

### Q2:
 Locate the expected-returns-replications.Rmd file in the vignettes directory. Refactor sections of this vignettes to replace functions from the plm package with the fitFfm or fitFfmDT functions associated with the FactorAnalytics package. This may include debugging upstream issues with merging data series, as well as reformatting data to match requirements of the new function arguments.

##### Thought
For Fama-Macbeth-plm and Fama-MacBeth,  they share a similar calculation procedure. First is to do a time-series regression. Secondly, doing a cross sectional regression.
Similar for Fama-French, we also need to perform time series regression first, then followed by a cross sectional regressin.

However, after checking ```help(fitFfm)``` , I found that it is specifically designed for cross-sectional regression.
When I try to google "fitFfm time series", I found "fitTsfm" is for time-series regression. Therefore, I decide to use fitTsf and fitFfm to realize Fama-French.

##### Solution
```
#Use fitTsfm to do time series regression
#Things worth mentioning: fitTsfm requires row column to be DATE.

test.data.third <- dcast(test.data, DATE ~ TICKER,  value.var = "EXC.RETURN")
test.data.third <- merge(test.data.third, ff3.data.monthly[c('DATE', 'MKT.RF','SMB','HML')], by='DATE')
row.names(test.data.third) <- test.data.third$DATE
test.data.third$DATE <- NULL
fit.test <- fitTsfm(asset.names = unique(factorDataSetDjia5Yrs$TICKER), factor.names = c('MKT.RF','SMB','HML'), data = test.data.third)
fit.test.coefs <- cbind(fit.test$alpha, fit.test$beta)
fit.test.coefs == betas
fit.test.coefs == beta.coefs


#Use fitFfm to do cross sectional regression

test.data.fourth  <-  data.frame(test.data[, c("DATE", "TICKER", "EXC.RETURN")], fit.test.coefs[, 2:4]) 
fit.test.second <- fitFfm(data=test.data.fourth, asset.var="TICKER", ret.var="EXC.RETURN", date.var="DATE", exposure.vars=exposure.vars, addIntercept=TRUE, lagExposures = FALSE)
fit.test.second.coefs <- data.frame(unique(test.data.fourth$DATE), fit.test.second$factor.returns)
colnames(fit.test.second.coefs) <- c('DATE', '(Intercept)', 'MKT.RF', 'SMB', 'HML')
row.names(fit.test.second.coefs) <- NULL
fit.test.second.coefs == gammas
fit.test.second.coefs == gamma.coefs
```


### Q3.
#### 3.1
How do you interpret the results of the new functions? 
##### Answer:
For dataset test.data, we first perform a time series regression to get beta.coefs. It shows for each ticker’s exc.return, eg. AA, while SMB and HML maintain the same, one unit change of MKT.RF causes 0.023844732 unit change of AA. While SMB and MKT.RF doesn’t change, one unit change of HML makes AA lose 0.0081158334. One unit change of SMB will make AA exc.return increase by 0.0067727223 unit. I also interpret it as the following. It extracts features of test.data, where we can observe MKT.RF, SMB, and HML ‘s affection. For instance, whether it is positive or negative, how much effect it has, etc. It ignores influence of date time and makes the data more readable. 

Then we perform a cross section regression to get gamma. It shows, for each day, for instance, 2008-01-01, while SMB and HML maintain the same, one unit change of MKT.RF makes Jan 1st’ exc.return decrease by 2.5019016. While SMB and MKT.RF doesn’t change, one unit change of HML makes Jan 1st’ exc.return increase 6.0815310. One unit change of SMB will make AA exc.return loose by 0.882210636 unit. When there is no outside influence(SMB, MKT.RF, HML are all zero), Jan 1st’s exc.return should be -0.2141849953.


#### 3.2
In addition, was there any repetitious code in the vignette that may be written as a function for future use? If so please include it as an example. 
##### Answer:
For repetitious code blocks, I find two shown as the following.

* **Function name**:```del_col(data, delete_column_names)``` - To remove columns by column name  
**Repetitious example**: 
```test.data$RF <- NULL
test.data.third$DATE <- NULL
```     
**Improvement**: In repetitious examples, it cost only one line to remove the unwanted column. 
However, if we want to delete two/more columns, we need to write two/more lines of code. Eg. data$val1<-NULL, data$var2<-NULL,In my solution, we can wrap them into a function which supports multiple columns’ deleting at the same time. In this way, we can only use one line.               
**Sample usage**: To delete Date, RF, TICKER at the same time.
    	```test.data = del_col(data, names = c(“Date”,” RF”, “TICKER”))```



* **Function name**:```appy_time_result_to cross()``` - Change time series data into cross sectional data 
by adding time-series coefs into origin data.   
**Repetitious example**: 
```test.data.input.second <- test.data.input[order(test.data.input[, 'DATE']), ]
test.data.input.second <- data.frame(
  test.data.input.second[, c("PERIOD.ID", "ASSET.ID", "DATE", "EXC.RETURN")], 
  betas[, 2:4]
)
```     
**Improvement**: There are two steps. First to sort by date, second is to combine betas and 
original data together. This procedure is similar at Fama-Macbeth plm and Fama-
Macbeth. Thus, I think we can use function to replace these two lines. Also, it will be useful because doing both time-series and cross sectional regression is common. Thus this can help a lot.       
**Sample usage**: ```appy_time_result_to cross(data.input, betas, exposure_names= c("PERIOD.ID", "ASSET.ID", "DATE", "EXC.RETURN"))```



#### 3.3
What data transformations or models might have benefited from writing unit tests? Please include examples for these as well.

##### Answer:
I believe every model can benefit from unit test. The importance lies in how to write complete testing.

Take the instance of From my point of view, we need to consider data type, such as int and float. For each parameter, we need to consider all its possible values. For instance, we need to test fit.method “LS", "WLS", "Rob", "W-Rob" case by case. Test z.score with value "none", "crossSection" and "timeSeries" one by one. 


At the same time, inside the model, it will check whether input data is in correct format. If not, it will throw error. In writing unit test, we can also include the ERROR cases to check whether it actually stops the program.




