# Data Exploration

## Overview
This section explores the data used for completing the final project of the Big Data Specialization by University of California San Diego on Coursera. The project focuses on developing recommendations for increasing revenue in a fictitious online game called "Catch the Pink Flamingo". 

The data for the project consists of 8 files with data on in-app purchases, ad clicks and game-specific information as well as 6 files with chat data.  The data were downloaded from the Coursera website.  All the data for the project were generated by a development team of data scientists.  The data were designed to simulate many aspects of game play and in-game user activity.  

The project entails classification analysis by fitting a decision tree and cluster analysis using k-means.  It also uses graph analytics to identify the chattiest users / teams, longest conversations and active user groups. 

The analysis in this project was performed using Python, Spark, KNIME and Neo4j.

This section is organized as follows:

**Overview**<br/>
**Preliminary Data Exploration**<br/>
**Data Preparation for Classification Analysis**<br/>
**Data Preparation for Cluster Analysis**<br/>
**Data Preparation for Graph Analytics**

The analysis is described in the [next section](https://eagronin.github.io/capstone-prepare/).

## Preliminary Data Exploration

As a first step, we load the file `buy-clicks.csv` described in the [previous section](https://eagronin.github.io/capstone-acquire/) into Splunk and output several summary ststistics and charts to understand the data.  We find that the total amount spent on buying in-app purchase items is $21,407, while the number of unique items available to be purchased is 6.  

Below is a histogram showing how many times each item is purchased:

![](https://github.com/eagronin/capstone-prepare/blob/master/the-number-of-times-each-item-is-purchased.png?raw=true)

We can see on the chart below how much money was made from each item:

![](https://github.com/eagronin/capstone-prepare/blob/master/revenue-generated-by-each-item.png?raw=true)

We note that the revenue was calculated as the sum of prices for each item multiplied by 2% which is the amount that Eglence, Inc. earns for each item sold.

We then filter the data to plot the total amount of money spent by the top ten users (ranked by how much money they spent):

![](https://github.com/eagronin/capstone-prepare/blob/master/amount-spent-by-the-top-ten-users.png?raw=true)

The following table shows the user id, platform, and hit-ratio percentage for the top three buying users:

Rank | User Id | Platform | Hit-Ratio (%)
--- | --- | --- | ---
1 | 2229 | iphone | 11.6%
2 | 12 | iphone | 13.1%
3 | 471 | iphone | 14.5%

Overall, the data contain information on 1,411 in-app purchases made by 546 users out of the total number of 2,393 users 

The 2 most expensive items (out of the total of 6 items available for purchase) generated 76.8% of revenue for Eglence, Inc.
Futher, amount spent by top 10 users is 8.96% of the total amount spent (1,919 / 21,407 * 100), while these users represent only 0.42% of all users (10 / 2, 393 * 100)

Based on this preliminary data exploration, it appears that identifying likely purchasers of the most expensive items and heaviest spenders for targeted marketing can increase revenue for Eglence, Inc.

## Data Preparation for Classification Analysis

In-app purchases of expensive items generate more revenue for Eglence, Inc. than purchases of inexpensive items. Therefore, it is important to identify users who are more likely to purchase expensive items and target such items to these users.  We call the users who tend to pucharaseexpensive items “HighRollers” and the users who tend to purchase inexpensive items “PennyPinchers”.  Big-ticket items are those with a price of more than $5.00, and inexpensive items are those that cost $5.00 or less.  

In this section we prepare the data for fitting a decision tree to make predictions which users are HighRollers and which ones are PennyPinchers based on the known attributes.  We are using `combined_data.csv` described in the [previous section](https://eagronin.github.io/capstone-acquire/) for this analysis.

A new categorical attribute was created to enable analysis of players as broken into 2 categories (HighRollers and PennyPinchers).  A screenshot of the attribute in KNIME follows:

![](https://github.com/eagronin/capstone-prepare/blob/master/attribute-creation.png?raw=true)

This new attribute was derived by binning avg_price into two categories: HighRollers (buyers of items that cost more than $5.00) and PennyPinchers (buyers of items that cost $5.00 or less). The attribute takes the value of 1 if avg_price is higher than $5.00 and zero otherwise.  The name of the new attribute is “highRollers”.  Of the 1411 samples in the dataset there are 575 HighRollers and 836 PennyPinchers.

The creation of this new categorical attribute was necessary because the goal of building the model is to predict whether the player belongs to one of the two categories (HighRollers or PennyPinchers) rather than predicting the expected purchase price (in which case a continuous feature, such as avg_price, would be used without splitting it into two categories).

The following attributes were filtered from the dataset for the following reasons:

Attribute | Rationale for Filtering
:--- |:---
userId | User ID has been removed from the dataset because it is arbitrarily assigned by the game and, as such, is irrelevant for determining whether the player is a HighRoller or a PennyPincher.
userSessionId | Session ID has been removed from the dataset because it is arbitrarily assigned by the game and, as such, is irrelevant for determining whether the player is a HighRoller or a PennyPincher.
avg_price | Average price has been removed from the dataset because it was used for creating the target.  Keeping avg_price in the dataset would create an illusion of strong predictive power of the model due to the relationship between avg_price and the target.  Moreover, avg_price is not going to be available for prediction in the real world, because we will need to make a prediction whether the player is a HighRoller or a PennyPincher before the player made a purchase.

The data was partitioned into train and test datasets.  The train data set was used to create the decision tree model.  The trained model was then applied to the test dataset.  This is important because training a model may result in overfitting, in which case the model will be fitted to reproduce the noise in the training sample and, as a result, will perform poorly when applied to making predictions of the target using new samples that have not been used in training the model.  As such, evaluation of predictive power of a model should be done using test data that has not been used in training the model.

When partitioning the data using sampling, it is important to set the random seed in order to be able to reproduce the results.  If the random seed is not set, then any random sampling performed while fitting the model will result in variation in the results each time the model is fitted (to the same training data), which will make the results not replicable.  

## Data Preparation for Cluster Analysis
Groups of users with different attributes (for example, experienced vs. inexperienced players) are likely to have differences in their tendency to make in-app purchases.  Therefore, revenue and profitability can be increased by choosing different strategies for targetingand setting fees for hosting in-app purchase items shown to users from different groups.

We selected 3 attributes for cluster analysis that breaks out users into distinct groups:

Attribute | Rationale for Selection 
:--- |:---
teamLevel | User experience as reflected in the teamLevel attribute could be a differentiator of user behavior.  For example, the beginners might be too focused on learning the game and not pay attention to the in-app purchase items.  To the contrary, more experienced users may express more interest to the in-app purchases.
accuracyRate (count_hits / count_gameclicks) | Accuracy rate reflects how well a user is playing the game.  It may depend on user experience, yet may also reflect user intrinsic qualities.  For example, users with a higher accuracy rate may be more capable individuals, have higher incomes and, as a result, may spend more money while playing the game.
Revenue (avg_price * count_buyId when avg_price and count_buyId are not NULL, and zero otherwise) | While revenue may correlate with the other two attributes above (teamLevel and accuracyRate), it has its own differentiating characteristics.  For example, irrespective of skills and experience, users may differ in their inclination to make in-app purchases.

The training data set used for this analysis is shown below (first 5 lines):

`Features = (teamLevel, accuracyRate, revenue)`, scaled using `StandardScaler()`

```
Row(features=DenseVector([-1.24, -0.31, -0.41])),
Row(features=DenseVector([-0.72, -0.68, -0.41])),
Row(features=DenseVector([-1.76, -0.39, -0.41])),
Row(features=DenseVector([-0.72, -0.36, 0.24])),
Row(features=DenseVector([-0.19, 1.15, 0.03])),
...
```

Dimensions of the training data set (rows x columns) : 3688 x 3

## Data Preparation for Graph Analytics
Chattier users, initiators of longer conversations and users who belowng to chattier teams and active user groups are likely to be more valuable, because of their potential to spread information to wider audiences.  As a result, Eglence, Inc. can increase its revenue by choosing the right marketing strategy to target such users, for example, showing the more expensive items to such users.  Even if these users are not going to buy these items, they may influence others in their networks to buy such items. 

We discussed the process of loading the chat data into Neo4j in the [previous section](https://eagronin.github.io/capstone-acquire/).  Below is a screenshot that shows a portion of the graph:

![](https://github.com/eagronin/capstone-prepare/blob/master/graph-screenshot.png?raw=true)

Next step: [Analysis](https://eagronin.github.io/capstone-analyze/)



 
