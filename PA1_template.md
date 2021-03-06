# Reproducible Research: Peer Assessment 1

Welcome to my Peer Review Assignment 1 for Reproducible Research class.

Make sure the working directory is set to the location of the activity.csv file.

First off the activity.csv file is read in.

```r
library("lattice")

datafile <- "activity.csv"
datafile <- paste0(getwd(), "/", datafile)

stepdata <- read.csv(datafile, header = TRUE, comment.char = "")
```

A column is added to combine the date and time fields into a single POSIX format
date number.
Additionally a column is added to hold a logical TRUE value if the
steps field is marked na, FALSE otherwise.


```r
stepdata$datetime <- as.POSIXct(paste(stepdata$date, stepdata$interval%/%100,
                                      stepdata$interval%%100),
                                "%Y-%m-%d %H %M", tz = "GMT")
stepdata$isna <- is.na(stepdata$steps)
```

##Mean Total Number of Steps per Day

tapply is used to sum the total steps for each day, na's are removed before
summing.
allnas is a flag that a day consists entirely of na's.

The following histogram shows the frequency (how many times) a day had a step
total within the bar's range. Days that had all na's are not used in the
histogram since they their sum is 0, which is not a valid count of activity for
the day.


```r
dailysteps <- tapply(stepdata$steps, stepdata$date, sum, na.rm = TRUE)
allnas <- tapply(stepdata$isna, stepdata$date, all)

# don't use the days with all na's, since the sum will be 0 rather then na
hist(dailysteps[!allnas], breaks = 10,
     main = "Histogram of Total Steps per Day", xlab = "Total Daily Steps")
```

![plot of chunk step_histogram](./PA1_template_files/figure-html/step_histogram.png) 

```r
cat("The mean steps per day is: ", mean(dailysteps))
```

```
## The mean steps per day is:  9354
```

```r
cat("The median steps per day is: ", median(dailysteps))
```

```
## The median steps per day is:  10395
```

```r
rm(allnas)               # housekeeping
```

##Average Daily Activity Pattern

tapply is once again used to calculate the mean number of steps taken for each
time interval across all the days. The maximum activity is noted both in the
graph and on a line of output.


```r
steps <- tapply(stepdata$steps, stepdata$interval, mean, na.rm = TRUE)

# extract the time from the names the tapply function assigned in variable steps
# 0001-01-01 is used to mark the date as arbitrary, but not the time info
time <- as.integer(names(steps))
timePOSIX <- as.POSIXct(paste("0001-01-01", time %/% 100, time %% 100),
                   "%Y-%m-%d %H %M", tz = "GMT")
timesteps <- data.frame(steps, time, timePOSIX)

rm(steps, time, timePOSIX)         # housekeeping

timetics = as.POSIXct(c("0001-01-01 00","0001-01-01 06","0001-01-01 12",
                          "0001-01-01 18","0001-01-01 24"), 
                         "%Y-%m-%d %H", tz = "GMT")

plot(timesteps$timePOSIX, timesteps$steps, type = "l", xaxt = "n",
     xlab = "Time of Day (hh:mm)", ylab = "5 Minute Step Average",
     main = "Average Daily Activity")
axis(1, at = timetics,
     labels = c("00:00","06:00","12:00","18:00","24:00"))

maxsteps <- which.max(timesteps$steps)
timeofmax <- timesteps$timePOSIX[maxsteps]
maxsteps <- timesteps$steps[maxsteps]

points(timeofmax, maxsteps, pch=1, col = "mediumblue")
text(timeofmax, maxsteps,
     paste0("(",format(timeofmax, "%H:%M"), ", ", round(maxsteps, 0), ")" ),
     pos=4 , cex = 0.67, col = "mediumblue")
```

![plot of chunk avg_daily_activity](./PA1_template_files/figure-html/avg_daily_activity.png) 

```r
cat("On average at ", format(timeofmax, "%H:%M"), " the maximum number of steps, ", round(maxsteps, 0), ", occurred.", sep = "")
```

```
## On average at 08:35 the maximum number of steps, 206, occurred.
```

##Imputing Missing Values

A seperate data.frame is created from the original. The mean daily activity
calculated above for each time interval is used to replace missing values in
the original data.
The code assumes that all 288 (24 * 60 / 5) time intervals are present for each
day. This assumption appears to hold for this activity file since
17568 lines / 288 intervals = 61 even.

As above the number of steps for each day is again computed including the 
imputed data and a histogram generated.


```r
cat("There are", sum(stepdata$isna), "missing step values in the dataset")
```

```
## There are 2304 missing step values in the dataset
```

```r
imputeddata <- stepdata

# to fill in missing data use the mean for that time step.
# the entire imputedsteps column is initialized using the mean data to fill it 
imputeddata$imputedsteps <- timesteps$steps

# if the data is not na, then the mean data is overwritten with the valid value
imputeddata$imputedsteps <- ifelse(imputeddata$isna, 
                                   imputeddata$imputedsteps, imputeddata$steps)

dailyimputsteps <- tapply(imputeddata$imputedsteps, imputeddata$date,
                          sum, na.rm = FALSE)
hist(dailyimputsteps, breaks = 10,
     main = "Histogram of Total Steps per Day", xlab = "Total Daily Steps")
```

![plot of chunk impute_missing](./PA1_template_files/figure-html/impute_missing.png) 

```r
cat("The mean steps per day is: ", mean(dailyimputsteps))
```

```
## The mean steps per day is:  10766
```

```r
cat("The median steps per day is: ", median(dailyimputsteps))
```

```
## The median steps per day is:  10766
```

##Differences in Weekday / Weekend Activity Patterns

The date is used to factor each day as a weekend or weekday.
A lattice plot comparing the weekend and weekday activity is then presented.


```r
# find day of week for each date, then change it to either weekday or weekend
imputeddata$weekday <- weekdays(imputeddata$datetime)
imputeddata$weekday <- ifelse(imputeddata$weekday %in% c("Saturday", "Sunday"),
                              "weekend", "weekday")
imputeddata$weekday <- as.factor(imputeddata$weekday)

# use tapply to find mean number of steps taken on weekdays and then on weekends
weekdaysteps <- tapply(imputeddata$imputedsteps[imputeddata$weekday == "weekday"],
                       imputeddata$interval[imputeddata$weekday == "weekday"],
                       mean)
weekendsteps <- tapply(imputeddata$imputedsteps[imputeddata$weekday == "weekend"],
                       imputeddata$interval[imputeddata$weekday == "weekend"],
                       mean)

timeimputed <- data.frame(timesteps$timePOSIX, weekdaysteps)
timeimputed$dayofweek <- as.factor("weekday")
colnames(timeimputed)[2] <- "steps"
timeimputed2 <- data.frame(timesteps$timePOSIX, weekendsteps)
timeimputed2$dayofweek <- as.factor("weekend")
colnames(timeimputed2)[2] <- "steps"
timeimputed <- rbind(timeimputed, timeimputed2)

colnames(timeimputed)[1] <- "timePOSIX"

xyplot(steps ~ timePOSIX | dayofweek, timeimputed, type = "l", 
        scales = list(x = list(at = timetics,
                      labels = c("00:00","6:00","12:00","18:00","24:00"))),
       layout = c(1,2),
       xlab = "Time of Day (hh:mm)", ylab = "Average Steps")
```

![plot of chunk weekday_weekend](./PA1_template_files/figure-html/weekday_weekend.png) 
