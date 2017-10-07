This is a brief exploratory analysis of data from a personal activity
monitoring device, found
[here](https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2Factivity.zip).
It collects data at 5 minute intervals throughout the day. This file
consists of two months of data from an anonymous individual during the
months of October and November, 2012 and includes the number of steps
taken in each 5 minute interval throughout each day.

### Loading and preprocessing the data

We will begin by loading in the data and taking a look at a sample of
it.

    activity <- read.csv('activity.csv')
    dim(activity)

    ## [1] 17568     3

    head(activity)

    ##   steps       date interval
    ## 1    NA 2012-10-01        0
    ## 2    NA 2012-10-01        5
    ## 3    NA 2012-10-01       10
    ## 4    NA 2012-10-01       15
    ## 5    NA 2012-10-01       20
    ## 6    NA 2012-10-01       25

We should convert the date column to the date class and add leading
zeroes to the interval number so that we can properly convert it to a
time later.

    activity$interval <- sprintf("%04d", activity$interval)
    activity$date <- as.Date(as.character(activity$date), "%Y-%m-%d")
    head(activity)

    ##   steps       date interval
    ## 1    NA 2012-10-01     0000
    ## 2    NA 2012-10-01     0005
    ## 3    NA 2012-10-01     0010
    ## 4    NA 2012-10-01     0015
    ## 5    NA 2012-10-01     0020
    ## 6    NA 2012-10-01     0025

It's worrying that there are so many NA entries for 'steps', but if we
look farther down, we see it does contain numeric data.

    activity[1000:1010,]

    ##      steps       date interval
    ## 1000     0 2012-10-04     1115
    ## 1001     0 2012-10-04     1120
    ## 1002   180 2012-10-04     1125
    ## 1003    21 2012-10-04     1130
    ## 1004     0 2012-10-04     1135
    ## 1005     0 2012-10-04     1140
    ## 1006     0 2012-10-04     1145
    ## 1007     0 2012-10-04     1150
    ## 1008     0 2012-10-04     1155
    ## 1009   160 2012-10-04     1200
    ## 1010    79 2012-10-04     1205

### What is mean total number of steps taken per day?

We can create a histogram to see the distribution of how many steps are
taken each day. Use 'tapply' to sum the number of steps over each day.

    steps_sum <- with(activity, tapply(steps, date, sum, na.rm = TRUE))
    hist(steps_sum, breaks = 10, xlab = 'Steps', main = 'Steps Taken Daily')

![](Project_1_files/figure-markdown_strict/unnamed-chunk-4-1.png)

And we can use basic statistical functions like 'mean' and 'median' on
our sum vector to get more information on the average number of steps
taken each day.

    mean(steps_sum, na.rm = TRUE)

    ## [1] 9354.23

    median(steps_sum, na.rm = TRUE)

    ## [1] 10395

### What is the average daily activity pattern?

What abaout the average number of steps in each interval across all
days? Calculating and creating a time-series plot of this gives a good
idea of the subject's daily routine. Sum steps with 'tapply' using the
interval as the factor this time and convert the array labels to a time.

    interval_mean <- with(activity, tapply(steps, interval, mean, na.rm = TRUE))
    plot(strptime(names(interval_mean), "%H%M"), interval_mean, type = 'l', xlab = 'Time', ylab = 'Steps', main = 'Mean Steps: October and November, 2012')

![](Project_1_files/figure-markdown_strict/unnamed-chunk-6-1.png)

It is then easy to find which interval contains the maximum number of
steps on average: 206.16 steps at 8:35 each day, which is index 104 of
interval\_mean.

    which.max(interval_mean)

    ## 0835 
    ##  104

    max(interval_mean)

    ## [1] 206.1698

### Imputing missing values

Returning to the subject of NA's, we should see how much of our data is
missing and what percentage that corresponds to.

    sum(is.na(activity$steps))

    ## [1] 2304

    mean(is.na(activity$steps))

    ## [1] 0.1311475

Thirteen percent sounds like a non-negligable amount, so we should
impute this missing data in some way. Here we will simply replace any
missing values by the mean over all days for that corresponding
interval. We loop over the whole 'steps' column for this operation and
replace NA's using the named array in which we stored the interval means
earlier.

    for(i in 1:nrow(activity)) {
        if(is.na(activity[i,'steps'])) {
            interval <- activity[i, 'interval']
            activity[i, 'steps'] <- interval_mean[as.character(interval)]
        }
    }

Now we take a look at the histogram again to see if this changed
anything. It does seem to have narrowed the distribution, this is to be
expected since we increased the number of values equal to the average of
each interval. It also lowered the number of days in the lowest bin;
this means that before that bin consisted of a large amount of days with
mostly or only NA values. It is probably more accurate to treat those as
average days, as we do here.

    steps_sum <- with(activity, tapply(steps, date, sum, na.rm = TRUE))
    hist(steps_sum, breaks = 10, xlab = 'Steps', main = 'Steps Taken Daily')

![](Project_1_files/figure-markdown_strict/unnamed-chunk-10-1.png)

### Are there differences in activity patterns between weekdays and weekends?

Finally, we want to compare activity levels during the weekend versus
during weekdays. First, we add a new column to our data that identifies
the type of day by using the 'weekdays' function and then reading the
output string.

    day <- weekdays(activity$date)
    activity$day_type <- ifelse(day == 'Saturday' | day == 'Sunday','Weekend', 'Weekday')

Now, we 'tapply' the mean over both day type and interval, then use the
'melt' function from the 'reshape2' package to reshape the data into a
form where the interval information is contained in a single column so
it can be more easily plotted.

    day_type_means <- with(activity, tapply(steps, list(day_type, interval), mean, na.rm = TRUE))
    library(reshape2)
    day_type_means <- melt(day_type_means)
    names(day_type_means) <- c('day_type', 'interval', 'avg_steps')
    day_type_means$interval <- sprintf("%04d", day_type_means$interval)
    head(day_type_means)

    ##   day_type interval  avg_steps
    ## 1  Weekday     0000 2.25115304
    ## 2  Weekend     0000 0.21462264
    ## 3  Weekday     0005 0.44528302
    ## 4  Weekend     0005 0.04245283
    ## 5  Weekday     0010 0.17316562
    ## 6  Weekend     0010 0.01650943

Finally, we can use 'ggplot2' to create our comparison plot.

    library(ggplot2)
    qplot(strptime(interval, "%H%M"), avg_steps, data = day_type_means, facets = .~day_type, geom = 'line', xlab = 'Time', ylab = 'Mean Steps Taken', main = 'Mean Steps: October and November, 2012') + scale_x_datetime(date_label = "%H:%M")

![](Project_1_files/figure-markdown_strict/unnamed-chunk-13-1.png)
