Content of the blog:

Introduction to Solar Power and need for its forecasting
Preliminary Analysis
Data Preprocessing and Manipulation
Train-test split
Missing Value Imputation
Exploratory Data Analysis
Handling outliers
Model Building & Predictions
DSM as per Indian Power Sector Regulations
Operational Challenges & Brainstorming
Introduction to Solar Power and need for its forecasting 🌞
Solar power is the conversion of energy from sunlight into electricity, either directly using photovoltaics (PV), or indirectly using concentrated solar power systems.

One of the most sustainable and competitive renewable energy sources is the solar photovoltaic (PV) energy which this blog shall discuss.


The main crucial and challenging issue in solar energy production is the intermittency of power generation due to weather conditions. In particular, a variation of the temperature and irradiance can have a profound impact on the quality of electric power production.

Hence, accurately forecasting the power output of PV modules in a short-term is of great importance for daily/hourly efficient management of power grid production, delivery, storage, as well as for decision-making on the energy markets.

So, let’s begin!

Solar Power Forecasting basically is predicting the solar generation for future time blocks based on forecasted weather parameters like Irradiance, ambient temperature, humidity, wind speed and other relevant parameters.
The dataset is taken from: Solar Power Generation Data | Kaggle.

We have 2 files:

Generation dataset
The power generation datasets are gathered at the inverter level- each inverter has multiple lines of solar panels attached to it.


Total no. of records = 69,000

2. Weather Sensor Data


Preliminary Analysis
Doing the basic stuff: Importing all necessary packages and reading data in csv.


So, here are our Generation and Weather datasets.

Basic points to notice from the description:


DC Power > AC Power. As stated earlier, PV modules generate DC power, it’s converted into AC using inverters for transmission purposes which amounts to some loss due to imperfect Conversion efficiency. Greater the difference between these two, more is the power loss.
High difference between 75th percentile and Max values of Ambient Temperature, Module Temperature and Irradiation. This indicates lesser presence of high values. Reasons could be cold weather, less solar hours, presence of outliers, etc.
Will explore more using visualizations later.


PLANT_ID and SOURCE_KEY is same for all records so can be dropped from both the datasets.
Also, no. of inverters is 22 in df_gen.
No null values are present in both datasets.
SOURCE_KEY is a unique Inverter ID so will replace it with Inverter No. for simplicity using mapping & then drop the SOURCE_KEY column in both datasets.

Daily yield and Total yield columns can be useful in Inverter-wise analysis like comparing the daily yields of inverter connected Solar modules, module fault analysis, maintenance/cleaning requirement, shadowing effect individually and Plant performance analysis but here our aim is to forecast generation, so will drop these too.

Now let’s check for missing values inverter-wise.


Inverters have data missing for different time stamps with a maximum difference of 904 time blocks. These missing values have to be imputed or removed so that each timestamp has values of all Inverters. Removing those rows may lead to loss of useful info so will go with suitable imputation.

Data Preprocessing and Manipulation
Right now, we have rows based on each inverter i.e. each timestamp is being repeated the number of inverters for which data is present for that particular time stamp(eg. suppose for 1st timestamp, only 12 inverters’ data is present so it will show 12 rows for that single time-stamp).


But we want our predictions to be on the plant level day wise divided into each 15 min timestamp. Hence the desired format is day-wise (timestamp-wise) rows from 00:00 to 24:00 with columns being the sum of all inverter values(for DC Power, AC Power…) for that timestamp as shown below.


For this, we shall first group our data inverter wise & store each group in a list. Then will merge each group with the next one using outer join on DATE_TIME column using ‘reduce’.


In order to retain maximum rows using outer join, a lot of null values have been introduced for uncommon timestamps between the inverters which we shall impute later. In similar way, will merge this Generation dataset with our Weather dataset to get a single dataset.

Will separate out DATE and TIME and make a new column named ‘BLOCK’ with each time BLOCK representing a 15 min interval using Pandas date_range function and will save in a dictionary. So, each day will have 96 time blocks from 00:00 as 1st to 23:45 as 96th. This will help in splitting during stratified training.


Finally our dataset is ready for analysis and imputation, but before that we’ll split it into train and test and keep our test dataset separate.

Train-Test split
Instead of going for random splitting, we will select last 3 days for test and rest as train for the sake of continuity which will help in our imputation strategy.


Thus, we have 2971 rows in train and 288 rows in test. Also, train dataset has a lot of missing values which we will impute in next segment.

Missing Data Imputation
For Imputation, first we need to get the essence of weather and generation data pattern.

Let’s plot for a random date!


It’s clear that the columns are continuous, ordered and possess some sequential property. Hence for filling a missing value, the previous and next time stamp values can be useful, that’s why we haven’t split the dataset randomly for train and test to preserve the sequences.

How to impute then?

Find the mean/median/mode for each time-block instead of population mean/median/mode from the training data and use for imputation in the test data in a block-wise manner.
Train a model only on non-missing values to later predict the missing values.
Both strategies would do good but when it comes to the changing nature of data in real world scenario as all weather parameters would vary with season(e.g. shifting of temperature, irradiation profile with time in winters), our training data will change periodically and hence the imputation values. So, will require to repeat strategy 1 & 2 again and again on new training data.

Also, strategy 2 being computationally expensive as well as time taking so is not recommended generally for productionization.

What to do now?

Spline Interpolation!

What it does is, it generates a polynomial between 2 nearest non-zero values to impute the missing values between them(polynomial order is to be specified).

Nature of columns is different. As its understood, Irradiation is a day only phenomenon and so the AC/DC Power generation, meaning their values would be zero for non-solar hours(00:00 to 06:00 and 1800 to 24:00) and so will impute with zero for these hours. The need for dealing it separately arises from a situation when there are all NaN values b/w say 5 pm of one day and 8 am of the next day. On applying spline imputation here, the duration of night hours will also get filled with non-zero values because both timestamps have non-zero values which we do not want.

Now for solar hours, will use linear polynomial for Irradiation, and degree 2 polynomial for AC/DC Power. For module temperature Linear spline would do okay. In short:

Irradiation(for solar hours), Module Temperature: Linear spline
AC/DC Power(non-solar hours), Irradiation(non-solar hours): Zero
AC/DC Power(solar hours): Polynomial spline(degree=2)
One may wonder why are we using degree 2 spline interpolation for AC/DC Power. A typical solar curve resembles an inverted parabola and hence the generation doesn’t vary linearly during solar hours. One more important reason for doing so is to introduce smoothness in the curve. This can be understood from the below comparison plot for missing value imputation using degree 1 & 2 spline for a particular day.



Degree 2 spline Imputation is much more smooth and near to the parabolic nature.

But why smoothness? → Wait for the last segment on DSM Management 😀

To codify what all imputation we discussed:

We have data prepared on the plant level now, i.e. all 22 inverter values have been summed up and scaled to MW from kW.

Now, we have our train & test data ready for more action.

Time for EDA now!

Exploratory Data Analysis
Our data is ready on the plant level now, so lets gain some useful insights.

Starting with Pair-plots.


Preliminary findings:

Strong linear relation between AC Power & Irradiation and Module Temperature & Irradiation.
Distributions of Irradiation, AC/DC Power are highly skewed to the right due to zero values for half of the day’s time being non-generating (6 pm-6 am), Ambient & Module Temperatures being slightly less skewed.
Increasing variation in Module Temperature with each degree rise in Ambient temperature. Variation indicates the possibility of influence by other weather parameters like humidity level, wind speed, precipitation, etc. also which are absent in dataset.
Presence of outliers in AC Power.


Skewness is also visible from the boxplots above(have ignored zero values while plotting)

Let’s find the correlations among the features. Not only in finding relations but this will help us in choosing the features for model building also.


More findings:

Correlation heatmap confirms our finding no. 1 of strong correlation between the variables.
Correlation b/w Irradiation & DC Power is slightly more than that of Irradiation & AC Power indicating loss in power during conversion. This can be further studied for inverter wise conversion analysis also(using previous df_train) as discussed earlier.
Comparatively less correlation b/w Ambient temperature & (AC Power|Irradiation) indicating less influence in predicting the generation.
Though it’s evident from the analysis and well known otherwise that Irradiation is the single most important factor for solar power generation, we’ll proceed with all features but DC Power for further course. As in the real-world scenario when predicting in real time for future time blocks, we may not find that much accurate forecasted weather data to completely rely on and over emphasis on those few features may affect the model performance significantly..

Handling Outliers
Outlier removal is a very important step in pre-processing. Presence of outliers may directly impact the performance of the model, more particularly in algorithms which work on distance based approaches like Linear Regression, k-Means, etc.

Let’s see percentile values of all features.


We’ll be using a simple approach for handling outliers. All points lying beyond the 99 percentile and below 1 percentile shall be replaced with the 99th and 1st percentile respectively. We’ll make a dictionary to save the 1st and 99th percentile values out of train data based on which outliers from both train and test data shall be replaced. Once saved, this dictionary can be loaded later for use.


Finally, our dataset is clean & completely ready to go for the most interesting part-> Model Building 😍

Model Building 🔨
We will start with different baseline models and choose the best with least error.

Performance Metric

Selection of Performance metric is a very important task.

For Regression problems, generally we use:

Mean Absolute Error(MAE), Mean Squared Error(MSE) and Root Mean Squared Error(RMSE)

We’ll be choosing RMSE over MAE and MSE. In this problem statement we want to penalize the bigger errors more than the small ones. As in MSE and RMSE, error terms are squared and hence loss gets more amplified due to squaring of bigger errors. In order to reduce this loss, the model tries to fit in a way that these big errors are reduced.

MAE gives very general idea of the error and treats all the errors as same. In the last segment of DSM Management, will get to know about the deviation bands and penalty which will validate our choice of error metric.

Training Strategy

For feeding balanced data based on time-period to the model, we’ll make a new column named BINS. Each row shall be assigned a bin based on its corresponding Block No. as below:

Block (0–12) => BIN 1

Block (13–24) => BIN 2

…

Block (85–96) => BIN 8

Let’s not forget that each Block represents a timestamp, so each bin shall correspond to a time period during a day. This is to ensure that each of our training folds get equal taste of all the bins.


Now, as we have our data divided into bins, the data is ready for training.

With the help of Scikit learn’s beautiful ‘Pipeline’ functionality to wrap different models for the ease of training, we have used here 7 different Regression algorithms with default parameters including a 3 layered Neural Network regressor.

Can’t forget to standardize. Features have varying scales and for algorithms such as linear regression, it is good to standardize the data before training as it helps in reaching the global minimum faster.

Now let’s go for finalizing the model.

When it comes to model selection and evaluation, k-fold Cross validation is a good approach. Going one step further, we’ll be using Stratified k-folds based on the BINS column. This will help us get the same distribution of bins per fold.

Model Evaluation and Selection using stratified k-Fold Cross validation

Split df_train into into 8 folds
Use 7 folds for training(xtrain, ytrain), 8th fold for validation(xvalid, yvalid)
Standardize the xtrain & xvalid generated in step 2
Fit xtrain, ytrain on the model
Predict on xvalid, find the RMSE value and store in a list
Repeat steps 1–5 for 8 iterations and find mean of the list having RMSE scores to get model mean RMSE for 8 iterations
Repeat steps 1–6 for each model in Pipeline list and compare the results

Okay, so Random forest Regressor has performed the best. Let’s go forward with this and tune its parameters.

Hyperparameter Optimization

We’ll be making a grid of hyperparameters using Randomized Search. Once it finds the best parameters, will train the model on that.


Making Predictions
Time to make the predictions finally! 😎

Remember we had kept last 3 days data for testing? Completely unseen by the model and free from any kind of ‘Data Leakage’. Missing value Imputation and Outlier Removal has been done separately on this, so lets predict.


Awesome! Though, the RMSE on Test set is slightly higher than what we had for Training data(1.7386) but seems good considering the less amount of training data. Training the model on more data will definitely improve our model.

Let’s plot the results.



High fluctuations in actuals can be observed in all the 3 days. Predictions(Orange) in plot 1 is on the higher side in the middle part, rest is covering well. Forecasts in Plot 2 & 3 are slightly underfitting the actual curve. But as I said, we have a margin of (+/-)7% (first deviation band) with no penalty.

2 abnormal dips in Actuals(blue) visible in Plot 2 & 3 are due to erroneous data(very less generation @ high Irradiation).

Our approach would be slightly different and in line with the Real World scenario of Indian Power market. Let me introduce the concept of DSM first.

Bonus: {Deviation Settlement Mechanism} 💰
Let’s get our hands dirty on the commercial & operational aspect of Forecasting & Scheduling of renewable power.

Due to the intermittent nature of renewable energy, forecasting and scheduling of power is essential to maintain the stability and safety of the electricity grid. State electricity regulatory commissions have issued forecasting and scheduling regulations that provide the mechanism by which developers shall be penalized in case they deviate from the forecasted generation beyond a certain level. This mechanism is called Deviation Settlement Mechanism (DSM). There are deviation bands specified by states which govern the mechanism for intra-state energy flow.

Consider the case of Gujarat state. There is no penalty for ≤7% deviation, above which its applicable.

Interesting to note is, the rate of penalty keeps on increasing as we cross Deviation bands further(meaning higher the deviation % more the penalty rate)as shown below.


Deviation settlement mechanism (DSM) and its impact | Deviation settlement (ceew.in)

For other states, deviation bands & penalty rates specified are different but increasing manner of deviation rates with deviation % is similar.

With this basic understanding of DSM, let’s consider a scenario

Case 1
2 Blocks deviation with 6% each: RMSE = 6, Penalty = No(as deviation is within 7%)

Case 2
2 Blocks deviation at 8% & 0% : RMSE = 5.6568, Penalty = Yes(as deviation >7%)

Though Case 2 has lesser RMSE compared to Case 1 but as per the regulations, it will incur penalty for 8% dev, while no penalty in Case 1.

This penalty difference will only magnify as deviation % jumps to upper deviation band.

Hence, in the context of DSM, its important to keep the deviation for each block within the threshold limit along with minimizing the overall RMSE/MSE.

Deviation & DSM calculation as per the Regulations.
Deviation formula specified is slightly different here. Instead of Actuals in the denominator, we’ve Available capacity of plant(25 MW here).

Deviation % = abs(Actual - Prediction)/Available capacity * 100

Have defined 2 functions here as per respective State DSM Regulations for Forecasting & Scheduling of Renewable Power:

dsm_code_1( ): For states such as Madhya Pradesh, Uttar Pradesh, Andhra Pradesh, Karnataka, etc. having 15, 25, 35 & above % deviation bands with rates of Rs. 0.50, 1.0, 1.50 per unit(kWh)
dsm_code_2( ): For states like Gujarat having 7, 15, 23& above % deviation bands with rates of Rs. 0.25, 0.50, 1.50 per unit(kWh)
These have 3 parameters as input: Predictions, Actual generation and Available capacity.

This will generate a block-wise deviation and DSM penalty report.


DSM Report is ready for Day 1. Similarly, we can generate for other days also. This DSM penalty is be paid to the concerned State Load Dispatch center for deviating from forecast.

Operational Challenges & Brainstorming
In this final section of the blog, would like to discuss about some operational challenges.

In real time, forecast/prediction is to be done on the forecasted weather parameters which already has some error associated and in addition, the unpredictability in weather only worsens the situation.

Especially in rainy weather, detecting the presence of moving clouds becomes extremely difficult leading to large variation in the weather parameters. Latency in receiving real time actual data from plant is another issue.

Next, there is a restriction on the number of changes/revisions allowed during a day for solar and wind, 8 & 14 respectively. Also, revision can be done only 4th time block onwards, i.e. revision becomes effective only after 45 mins delay from the time of its submission after which you cannot change for further 1.5 hrs in some states.

All these challenges require the model to be robust, an overfit model will fail miserably. Therefore, numerous arrangements such as moving averages, exponential weighted moving averages (for gaining momentum from previous time stamps and smoothening), bins specific modifications, and different plant specific feature engineering are done to fit the scenario.

Such challenges have widen the scope for research in the field of weather prediction using AI & ML & for its application in renewable energy enormously.

With governments determined to reduce the carbon footprint for meeting their renewable energy targets, introduction of new segments, policies and markets for Green energy, a greener and brighter future for the world is inevitable.
