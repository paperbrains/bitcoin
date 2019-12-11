---
title:            "You can't trust Bitcoin's timestamps"
date:             2019-13-10T06:46:19 # XML Schema Date/Time
last_modified_at: 2019-13-10T06:46:19 # last page modified date/time
image:            /assets/images/empty-blocks/empty_blocks_miner.png # /assets/images/empty-blocks.jpg
excerpt:          "While looking at Bitcoin's interblock time data, I observed that there are thousands of blocks that are timestamped in non-chronological order. This post explored that peculiarity." # Optional for overring content excerpt
categories:       ["Blocks"] # ["category1"] - best is to have one category in a post
tags:             ["Onchain"] # ["tag1", "tag2", "tag3"] - you can have several post tags
---
While looking at Bitcoin's interblock time data, I observed that there are thousands of blocks that are timestamped in non-chronological order. This post explored that peculiarity.

First, we import the necessary libraries.

```python
import blocksci
import pandas as pd
import seaborn as sns
import numpy as np
import matplotlib as mpl
import matplotlib.pyplot as plt
from matplotlib.ticker import AutoMinorLocator
import matplotlib.style as style
from matplotlib.widgets import TextBox
from matplotlib import rcParams
from matplotlib.offsetbox import AnchoredText
from datetime import datetime, timedelta
chain = blocksci.Blockchain("/home/ubuntu/bitcoin")
```

We now create a dataframe containing the interblock time of each block.


```python
df = pd.DataFrame(list(range(0,len(chain))))
df = chain.heights_to_dates(df)
df.columns = ["block_index"]
df["time"] = df.index
df["time_prev_block"]=df["time"].shift(1)
df["time_delta"]=df.time - df.time_prev_block
df = df.dropna()
df = df.drop(pd.Timestamp('2009-01-09 02:54:25'))
df["td_mins_float"]= df.time_delta/timedelta(minutes=1)
df["rolling_month_mean"] = df.td_mins_float.rolling(window=4032).mean()

```

We create a chart of average interblock time, as a monthly moving average.


```python
sns.set()   # Reset default params
sns.set(context ="paper")   # Enable efficient scaling of fonts. Choose from 'poster' , 'paper' 'talk'
style.use('fivethirtyeight')
sns.set_palette("deep")

# Choose the font family e.g sans-serif. Modify the family below
mpl.rcParams['font.sans-serif']=['Helvetica']
mpl.rc('font', family='sans-serif', stretch="semi-condensed" )
mpl.rc('axes', facecolor='#F2DFCE', edgecolor='grey', titlepad=27, labelpad=6, labelsize='small', linewidth=0.6, titleweight="bold")
mpl.rc('lines', linewidth=1.4, linestyle='-')
mpl.rc('patch', force_edgecolor=True, linewidth=0)
mpl.rc('xtick.major', size=4, width=1 )
mpl.rc('xtick.minor', size=2, width=0.6,  visible=True)
mpl.rc('xtick', bottom=True, color="grey")
mpl.rc('axes.spines', right=False, top=False, left=False)
mpl.rcParams['axes.grid.axis'] = 'y'
mpl.rc('grid', color="white", linewidth=0.57)
mpl.rc('figure', figsize=(8.5333,4.8), facecolor='#F2DFCE' , dpi=600, titleweight='light', titlesize='medium')


# Creating my figure and setting up first sublot within the figure
fig, ax1 = plt.subplots()
plt.subplots_adjust(bottom=0.11, top=0.91)

# Plotting the line via OOP method
ax1.plot(df.time, df["rolling_month_mean"])
ax1.set_xlabel("Year")
ax1.set_ylabel("Minutes between blocks")
ax1.set_title("Average interblock time, as per timestamp", loc="left")
ax1.xaxis.set_minor_locator(AutoMinorLocator(2))
plt.suptitle("Monthly moving average", ha='left', va='bottom', x=0.082, y=0.913)
plt.setp(ax1.get_xticklabels(), color="black")

# Adding the caption at the bottom (which is its own subplot in the figure, appended below the first)
axbox = plt.axes([0.05, 0, 0.9, 0.02], yticklabels=[], xticklabels=[], xticks=[], frame_on=False)
axbox.grid(False)
axbox.add_artist(AnchoredText("@travric", frameon=False, loc="upper right", pad=0, prop=dict(fontsize="8", fontweight="bold")))
axbox.add_artist(AnchoredText("Source: BlockSci", frameon=False, loc="upper left", pad=0, prop=dict(fontsize="8")))
plt.show()
```


![Interblock time mean](/assets/images/negative-interblock-time\Time_Betwen_blocks_rolling_mean.png)


It is a common misconception that Bitcoin's average interblock time is 10 minutes. For any given difficulty period, as the hash rate adjusts upwards, blocks are produced faster. Over the last 10 years, hash rate has trended upwards, hence the average block time has been only 9 minutes and 27 seconds.

It would be worthwhile plotting the distribution of interblock time.


```python
mpl.rc('patch', force_edgecolor=True, linewidth=0.7)

fig, ax1 = plt.subplots()
plt.subplots_adjust(bottom=0.11, top=0.91)

# Creating my x,y dataframe
df_chart = df[["time","rolling_month_mean"]]

# Plotting the line via OOP method
sns.distplot(df_chart.rolling_month_mean[df_chart.rolling_month_mean.notnull()], ax=ax1, bins=30)
ax1.set_xlabel("Minutes between blocks")
ax1.set_title("Interblock time", loc="left")
plt.suptitle("Bitcoin's Poisson distribution", ha='left', va='bottom', x=0.08, y=0.93)
plt.setp(ax1.get_xticklabels(), color="black")

# Adding the caption at the bottom (which is its own subplot in the figure, appended below the first)
axbox = plt.axes([0.05, 0, 0.9, 0.02], yticklabels=[], xticklabels=[], xticks=[], frame_on=False)
axbox.grid(False)
axbox.add_artist(AnchoredText("@travric", frameon=False, loc="upper right", pad=0, prop=dict(fontsize="8", fontweight="bold")))
axbox.add_artist(AnchoredText("Source: BlockSci", frameon=False, loc="upper left", pad=0, prop=dict(fontsize="8")))
plt.show()
```


![Interblock Time Distribution](/assets/images/negative-interblock-time\Bitcoin Poisson Distribution.png)


Interestingly, there are thousands of blocks with a reported negative interblock time. That is, they are timestamped in non-chronological order. The second dataframe contains the frequency of non-chronological blocks per month.


```python
df2 = df[df.time_delta<timedelta(seconds=0)]
df2 = df2["time"].resample('M').count()
df2 = pd.DataFrame(df2).rename(columns={"time":"frequency"})
```


```python
mpl.rc('patch', force_edgecolor=True, edgecolor='C0', linewidth=1.32)

# Creating my figure and setting up first sublot within the figure
fig, ax1 = plt.subplots()
plt.subplots_adjust(bottom=0.11, top=0.91)

# Plotting the line via OOP method
ax1.bar(x=df2.index, height=df2.frequency)
ax1.set_xlabel("Month")
ax1.set_title("Blocks with non-chronological timestamp", loc="left")
ax1.xaxis.set_minor_locator(AutoMinorLocator(2))
plt.suptitle("Frequency of occurance, per month", ha='left', va='bottom', x=0.082, y=0.913)
plt.setp(ax1.get_xticklabels(), color="black")

# Adding the caption at the bottom (which is its own subplot in the figure, appended below the first)
axbox = plt.axes([0.05, 0, 0.9, 0.02], yticklabels=[], xticklabels=[], xticks=[], frame_on=False)
axbox.grid(False)
axbox.add_artist(AnchoredText("@travric", frameon=False, loc="upper right", pad=0, prop=dict(fontsize="8", fontweight="bold")))
axbox.add_artist(AnchoredText("Source: BlockSci", frameon=False, loc="upper left", pad=0, prop=dict(fontsize="8")))
plt.show()
```


![Negative Interblock Frequency](/assets/images/negative-interblock-time\Blocks with non-chronological timestamp.png)


The occurance of non-chronological blocks is possible due to the fact that there is no central party to authorise what the true time is. The protocol rejects blocks with a timestamp earlier than the median of the timestamps from the previous 11 blocks or later than 2 hours after the current network time. The frequency of non-chronological timestamped blocks has decreased substantially in recent years due to the deployment of BIP 113 in March 2016. One attribute of the BIP was a move to median time-past as the reference point for lock-time calculations rather than the last block time.


---
