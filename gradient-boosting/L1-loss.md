# Gradient boosting: Heading in the right direction

\author{[Terence Parr](http://parrt.cs.usfca.edu) and [Jeremy Howard](http://www.fast.ai/about/#jeremy)}

In our previous article, <a href="L2-loss.html">Gradient boosting: Distance to target</a>, our model took steps towards the target $\vec y$ based upon the residual vector, $\vec y-F_m(X)$, which includes the magnitude not just the direction of $\vec y$ from our current composite model's prediction, $F_m(X)$. The residual vector makes composite models converge rapidly towards $\vec y$.  The negative, of course, is that using the magnitude makes the composite model chase outliers.   This occurs because mean computations are easily skewed by outliers and our regression tree stubs yield predictions using the mean of all target values in a leaf.  For noisy target variables, it makes more sense to step merely in the *direction* of $\vec y$ from $F_m$ rather than the magnitude and direction. 

This brings us to the second commonly-used vector with gradient boosting, which we can call the *direction vector*: $sign(y_i-F_m(\vec x_i))$. The direction vector elements are either -1, 0, or +1, one value for each observation $\vec x_i$.   No matter how distant the true target is from our current prediction, the vector used to take steps towards the target is just direction info without the magnitude.  This direction vector is a bit weird, though, because the vector elements are limited to -1, 0, or +1.  In two dimensions, such vectors can only point at multiples of 45 degrees (0, 45, 90, 135, ...) and so the direction vector will rarely point directly at the target. Just to be clear, the direction vector is not the unit vector in the direction of $y_i-F_m(\vec x_i)$.

If there are outliers in the target variable that we cannot remove, using just the direction is better than both direction and magnitude. We'll show in <a href="descent.html">Gradient boosting performs gradient descent</a> that using $sign(y-F_m(\vec x_i))$ as our step vector leads to a solution that optimizes the model according to the mean absolute value (MAE) or $L_1$  *loss function*: $\sum_{i=1}^{N} |y_i - F_M(\vec x_i)|$ for $N$ observations. 

Optimizing the MAE means we should start with the median, not the mean, as our initial model, $f_0$, since the median of $y$ minimizes the $L_1$ loss. (The median is the best single-value approximation of $L_1$ loss.)  Other than that, let's assume that the same recurrence relations will work for finding composite models, $F_m(\vec x)$, as we used for the residual vector version in the last article:

\latex{{
\begin{eqnarray*}
F_0(\vec x) &=& f_0(\vec x)\\
F_m(\vec x) &=& F_{m-1}(\vec x) + w_m \Delta_m(\vec x)\\
\end{eqnarray*}
}}

Recall that $F_m(\vec x)$ yields a predicted value, $y_i$, but $F_m(X)$ yields a predicted target vector, $\vec y$, one value for each $\vec x_i$ feature row-vector in matrix $X$. Here is the rental data again along with the initial $F_0$ model and the first direction vector:

\latex{{
{\small
\begin{tabular}[t]{rrrr}
{\bf sqfeet} & {\bf rent} & $F_0$ & $sign(\vec y$-$F_0)$ \\
\hline
700 & 1125 & 1150 & -1 \\
750 & 1150 & 1150 & 0 \\
800 & 1135 & 1150 & -1 \\
900 & 1300 & 1150 & 1 \\
950 & 1350 & 1150 & 1 \\
\end{tabular}
}
}}


<pyeval label="examples" hide=true>
import pandas as pd
import matplotlib.pyplot as plt
from matplotlib import rc
import matplotlib
import numpy as np
from scipy.optimize import minimize_scalar
from sklearn.metrics import mean_squared_error, mean_absolute_error
#rc('text', usetex=True)
matplotlib.rcParams['mathtext.fontset'] = 'cm'
matplotlib.rcParams['mathtext.rm'] = 'serif'
matplotlib.rc('xtick', labelsize=13) 
matplotlib.rc('ytick', labelsize=13) 

bookcolors = {'crimson': '#a50026', 'red': '#d73027', 'redorange': '#f46d43',
              'orange': '#fdae61', 'yellow': '#fee090', 'sky': '#e0f3f8', 
              'babyblue': '#abd9e9', 'lightblue': '#74add1', 'blue': '#4575b4',
              'purple': '#313695'}

def draw_vector(ax, x, y, dx, dy, yrange):
    ax.plot([x,x+dx], [y,y+dy], c='r', linewidth=.8)
    ay = y+dy
    yrange *= 0.03
    ad = -yrange if dy>=0 else yrange
    ax.plot([x+dx-4,x+dx], [ay+ad,ay], c='r', linewidth=.8)
    ax.plot([x+dx,x+dx+4], [ay,ay+ad], c='r', linewidth=.8)
	
def data():
    df = pd.DataFrame(data={"sqfeet":[700,950,800,900,750]})
    df["rent"] = pd.Series([1125,1350,1135,1300,1150])
    df = df.sort_values('sqfeet')
    return df

df = data()

def stub_predict(x_train, y_train, split):
    left = y_train[x_train<split]
    right = y_train[x_train>split]
    lmean = np.mean(left)
    rmean = np.mean(right)    
#     lw,rw = w
    lw,rw = 1,1
    return np.array([lw*lmean if x<split else rw*rmean for x in x_train])

eta = 1.0
splits = [None,850, 850, 725] # manually pick them
w = [None, (20,100), (5,30), (5,20)]
stages = 4

def boost(df, xcol, ycol, splits, eta, stages):
    """
    Update df to have direction_i, delta_i, F_i.
    Return MSE, MAE
    """
    f0 = df[ycol].median()
    df['F0'] = f0

    for s in range(1,stages):
        # print("Weight", w[s])
        df[f'dir{s}'] = np.sign(df[ycol] - df[f'F{s-1}'])
        df[f'delta{s}'] = stub_predict(df[xcol], df[f'dir{s}'], splits[s])
        df[f'wdelta{s}'] = df[f'dir{s}'] * 30
        df[f'F{s}'] = df[f'F{s-1}'] + eta * df[f'wdelta{s}']

    mse = [mean_squared_error(df[ycol], df['F'+str(s)]) for s in range(stages)]
    mae = [mean_absolute_error(df[ycol], df['F'+str(s)]) for s in range(stages)]
    return mse, mae

mse,mae = boost(df, 'sqfeet', 'rent', splits, eta, stages)
df['deltas'] = df[['delta1','delta2','delta3']].sum(axis=1) # sum deltas
</pyeval>

Visually, we can see that the direction vector has components pointing in the right direction of the true target:
 
<pyfig label=examples hide=true width="32%">
f0 = df.rent.median()
fig, axes = plt.subplots(nrows=1, ncols=1, figsize=(4, 3.5), sharey=True)

ax = axes
line1, = ax.plot(df.sqfeet,df.rent,'o', linewidth=.8, markersize=4, label="$y$")
# fake a line to get smaller red dot
line2, = ax.plot([0,0],[0,0], c='r', markersize=4, label=r"$sign(y-f_0({\bf x}))$", linewidth=.8)
ax.plot([df.sqfeet.min()-10,df.sqfeet.max()+10], [f0,f0],
         linewidth=.8, linestyle='--', c='k')
ax.set_xlim(df.sqfeet.min()-10,df.sqfeet.max()+10)
ax.set_ylim(df.rent.min()-10, df.rent.max()+20)
ax.text(815, f0+10, r"$f_0({\bf x})$", fontsize=18)

ax.set_ylabel(r"Rent ($y$)", fontsize=14)
ax.set_xlabel(r"SqFeet (${\bf x}$)", fontsize=14)

# draw arrows
for x,y,yhat in zip(df.sqfeet,df.rent,df.F0):
    draw_vector(ax, x, yhat, 0, np.sign(y-yhat)*2, df.rent.max()-df.rent.min())
    
ax.legend(handles=[line2], fontsize=16,
          loc='upper left', 
          labelspacing=.1,
          handletextpad=.2,
          handlelength=.7,
          frameon=True)

plt.tight_layout()
plt.show()
</pyfig>

But, without the distance to the target as part of our direction vector, the $w_m (\hat y - F_{m-1}(X))$ steps towards $y$ would move very slowly unless we cranked up the $w_m$ weights. Unfortunately, if we crank up the weights arbitrarily, the stages of boosting might oscillate around but never reach an accurate prediction. For example, if we set $w_m = 30$ and look at the weighted direction vectors for a few boosting stages, we see some $\hat y_i$ converging ($\vec x >= 900$) to $y_i$ and some $\hat y_i$ oscillating up and down ($\vec x < 900$). (We're using the $sign$, as if we had a perfect $\Delta$ predictor, in order to focus on how the weights affect movement of $\hat{\vec y}$.)

<pyfig label=examples hide=true width="90%">
f0 = df.rent.median()
fig, axes = plt.subplots(nrows=1, ncols=3, figsize=(11, 4), sharey=True)

# NUMBER 1

ax = axes[0]
ax.set_ylabel(r"Rent", fontsize=14)
line1, = ax.plot(df.sqfeet,df.rent,'o', linewidth=.8, markersize=4, label="$y$")
# fake a line to get smaller red dot
line2, = ax.plot([0,0],[0,0], c='r', markersize=4, label=r"$w_1 sign(y-f_0({\bf x}))$", linewidth=.8)
ax.plot([df.sqfeet.min()-10,df.sqfeet.max()+10], [f0,f0],
         linewidth=.8, linestyle='--', c='k')
ax.set_xlim(df.sqfeet.min()-10,df.sqfeet.max()+10)
ax.set_ylim(df.rent.min()-10, df.rent.max()+20)
ax.text(815, f0+10, r"$f_0({\bf x})$", fontsize=18)

ax.set_xlabel(r"SqFeet", fontsize=14)

# draw arrows
for x,y,yhat,d in zip(df.sqfeet,df.rent,df.F0,df.wdelta1):
    draw_vector(ax, x, yhat, 0, d, df.rent.max()-df.rent.min())

ax.legend(handles=[line2], fontsize=16,
          loc='upper left', 
          labelspacing=.1,
          handletextpad=.2,
          handlelength=.7,
          frameon=True)

# NUMBER 2

ax = axes[1]
line1, = ax.plot(df.sqfeet,df.rent,'o', linewidth=.8, markersize=4, label="$y$")
# fake a line to get smaller red dot
line2, = ax.plot([0,0],[0,0], c='r', markersize=4, label=r"$w_2 sign(y-F_1({\bf x}))$", linewidth=.8)
ax.plot([df.sqfeet.min()-10,df.sqfeet.max()+10], [f0,f0],
         linewidth=.8, linestyle='--', c='k')
ax.set_xlim(df.sqfeet.min()-10,df.sqfeet.max()+10)
ax.set_ylim(df.rent.min()-10, df.rent.max()+20)
ax.text(815, f0+10, r"$f_0({\bf x})$", fontsize=18)

ax.set_xlabel(r"SqFeet", fontsize=14)

# draw arrows
for x,y,yhat,d in zip(df.sqfeet,df.rent,df.F1,df.wdelta2):
    draw_vector(ax, x, yhat, 0, d, df.rent.max()-df.rent.min())
    
# ax.text(710,1250, "$m=1$", fontsize=18)

ax.legend(handles=[line2], fontsize=16,
          loc='upper left', 
          labelspacing=.1,
          handletextpad=.2,
          handlelength=.7,
          frameon=True)

# NUMBER 3

ax = axes[2]
line1, = ax.plot(df.sqfeet,df.rent,'o', linewidth=.8, markersize=4, label="$y$")
# fake a line to get smaller red dot
line2, = ax.plot([0,0],[0,0], c='r', markersize=4, label=r"$w_3 sign(y-F_2({\bf x}))$", linewidth=.8)
ax.plot([df.sqfeet.min()-10,df.sqfeet.max()+10], [f0,f0],
         linewidth=.8, linestyle='--', c='k')
ax.set_xlim(df.sqfeet.min()-10,df.sqfeet.max()+10)
ax.set_ylim(df.rent.min()-10, df.rent.max()+20)
ax.text(815, f0+10, r"$f_0({\bf x})$", fontsize=18)

ax.set_xlabel(r"SqFeet", fontsize=14)

# draw arrows
for x,y,yhat,d in zip(df.sqfeet,df.rent,df.F2,df.wdelta3):
    draw_vector(ax, x, yhat, 0, d, df.rent.max()-df.rent.min())
    
# ax.text(710,1250, "$m=1$", fontsize=18)

ax.legend(handles=[line2], fontsize=16,
          loc='upper left', 
          labelspacing=.1,
          handletextpad=.2,
          handlelength=.7,
          frameon=True)

plt.tight_layout()
plt.show()
</pyfig>

A weight of 30 is just too coarse to allow tight convergence to $\vec y$ across all $y_i$.  When using the residual vector, each data point gets a weight tailored to its distance to the target.  The problem we have with the direction vector is that a single weight across all $\hat y_i$ only works if it's very small. But, that means very slow convergence. So, the solution is to use a different weight for each group of similar feature vectors.  Mathematically, that means moving the weight term to a parameter of the weak models and making it a vector $\vec w_m$ for each stage, $m$:

\[
F_m(\vec x) = F_{i-1}(\vec x) + \Delta_m(\vec x; \vec w_m)\\
\]

With regards to weak models based upon regression trees stubs, each stub leaf gets its own weight. If we let $\vec w_1=[20,100]$, $\vec w_2=[5,30]$, and $\vec w_3=[5,20]$, then we get the following table of partial results:

<!-- Separate weight per leaf -->
<pyeval label=examples hide=true>
def stub_predict(x_train, y_train, split):
    left = y_train[x_train<split]
    right = y_train[x_train>split]
    lmean = np.mean(left)
    rmean = np.mean(right)    
#     lw,rw = w
    lw,rw = 1,1
    return np.array([lw*lmean if x<split else rw*rmean for x in x_train])

eta = 1.0
splits = [None,850, 850, 725] # manually pick them
w = [None, (20,100), (5,30), (5,20)]
stages = 4

def boost(df, xcol, ycol, splits, eta, stages):
    """
    Update df to have direction_m, delta_m, F_m.
    Return MSE, MAE
    """
    f0 = df[ycol].median()
    df['F0'] = f0

    for s in range(1,stages):
        # print("Weight", w[s])
        df[f'dir{s}'] = np.sign(df[ycol] - df[f'F{s-1}'])
        df[f'delta{s}'] = stub_predict(df[xcol], df[f'dir{s}'], splits[s])
        weights = np.array([w[s][0] if x<splits[s] else w[s][1] for x in df[xcol]])
        df[f'wdelta{s}'] = df[f'delta{s}'] * weights
        df[f'F{s}'] = df[f'F{s-1}'] + eta * df[f'wdelta{s}']

    mse = [mean_squared_error(df[ycol], df['F'+str(s)]) for s in range(stages)]
    mae = [mean_absolute_error(df[ycol], df['F'+str(s)]) for s in range(stages)]
    return mse, mae

mse,mae = boost(df, 'sqfeet', 'rent', splits, eta, stages)
df['deltas'] = df[['delta1','delta2','delta3']].sum(axis=1) # sum deltas
</pyeval>

<!--
<pyeval label="examples" hide=true>
#print(df)
# manually print table in python
# for small phone, make 2 tables
o = ""
for i in range(len(df)):
    o += " & ".join([f"{v:.2f}" for v in df.iloc[i,:][['sqfeet','rent','F0','dir1']]]) + "\\" + "\n"

for i in range(len(df)):
    o += " & ".join([f"{v:.2f}" for v in df.iloc[i,4:15]]) + r"\\"+ "\n"

o = o.replace(".00", "")
print(o)
</pyeval>
-->

\latex{{
{\small
\setlength{\tabcolsep}{0.5em}
\begin{tabular}[t]{rrrrrrrrrrr}
$\Delta_1$ & $\Delta_1(\vec x$;$\vec w_1)$ & $F_1$ & $\vec y$-$F_1$ & $\Delta_2$ & $\Delta_2(\vec x$;$\vec w_2)$ & $F_2$ & $\vec y$-$F_2$ & $\Delta_3$ & $\Delta_3(\vec x$;$\vec w_3)$ & $F_3$\\
\hline
-0.67 & -13.33 & 1136.67 & -1 & -0.33 & -1.67 & 1135 & -1 & -1 & -5 & 1130\\
-0.67 & -13.33 & 1136.67 & 1 & -0.33 & -1.67 & 1135 & 1 & 0.75 & 15 & 1150\\
-0.67 & -13.33 & 1136.67 & -1 & -0.33 & -1.67 & 1135 & 0 & 0.75 & 15 & 1150\\
1 & 100 & 1250 & 1 & 1 & 30 & 1280 & 1 & 0.75 & 15 & 1295\\
1 & 100 & 1250 & 1 & 1 & 30 & 1280 & 1 & 0.75 & 15 & 1295\\
\end{tabular}
}
}}

Visually, the unweighted direction vectors for the first three stages look like the following.

<pyfig label=examples hide=true width="90%">
def draw_stub(ax, x_train, y_train, y_pred, split, stage, locs):
    line1, = ax.plot(x_train, y_train, 'o',
                     markersize=4,
                     label=f"$sign(y-F_{stage-1})$")
    label = r"$\Delta_"+str(stage)+r"({\bf x})$"
    left = y_pred[x_train<split]
    right = y_pred[x_train>split]
    lmean = np.mean(left)
    rmean = np.mean(right)
    line2, = ax.plot([x_train.min()-10,split], [lmean,lmean],
             linewidth=.8, linestyle='--', c='k', label=label)
    ax.plot([split,x_train.max()+10], [rmean,rmean],
             linewidth=.8, linestyle='--', c='k')
    ax.plot([split,split], [lmean,rmean],
             linewidth=.8, linestyle='--', c='k')
    ax.plot([x_train.min()-10,x_train.max()+10], [0,0],
             linewidth=.8, linestyle=':', c='k')
    ax.legend(handles=[line1,line2], fontsize=15,
              loc=locs[stage-1], 
              labelspacing=.1,
              handletextpad=.2,
              handlelength=.7,
              frameon=True)

def draw_residual(ax, df, stage):
    for x,d0,delta in zip(df.sqfeet,df[f'dir{stage}'],df[f'delta{stage}']):
#         print(x, d0, delta)
        draw_vector(ax, x, d0, 0, delta-d0, 2)
#         if delta-d0!=0:
#             ax.arrow(x, d0, 0, delta-d0,
#                       fc='r', ec='r',
#                       linewidth=0.8,
#                      )

fig, axes = plt.subplots(nrows=1, ncols=3, figsize=(11, 3.5), sharey=True, sharex=True)

ax = axes[0]
axes[0].set_ylabel(r"Direction in {-1,0,1}", fontsize=14)
axes[0].set_yticks([-1,0,1])
for a in range(3):
    axes[a].set_xlabel(r"SqFeet", fontsize=14)
    axes[a].set_xlim(df.sqfeet.min()-10,df.sqfeet.max()+10)

locs = ['upper left','lower right','lower right']
draw_stub(axes[0], df.sqfeet, df.dir1, df.delta1, splits[1], stage=1, locs=locs)
draw_residual(axes[0], df, stage=1)

draw_stub(axes[1], df.sqfeet, df.dir2, df.delta2, splits[2], stage=2, locs=locs)
draw_residual(axes[1], df, stage=2)

draw_stub(axes[2], df.sqfeet, df.dir3, df.delta3, splits[3], stage=3, locs=locs)
draw_residual(axes[2], df, stage=3)

plt.tight_layout()
        
plt.savefig('/tmp/t.svg')
plt.show()
</pyfig>

The $\Delta_m$ weak models have a hard time predicting the three rent prices on the left, as you can see. Here are the stubs that generate those dashed lines:

<img src="images/stubs-mae.svg" width="90%">

Here are the weighted $\Delta_m$ prediction vectors for first three stages that illustrates how using two different weights for the same model allows much more control over steps to the target.

<pyfig label=examples hide=true width="90%">
f0 = df.rent.median()
fig, axes = plt.subplots(nrows=1, ncols=3, figsize=(11, 4), sharey=True)

def draw_stage_residual(ax, df, stage):
    for x,d0,delta in zip(df.sqfeet,df[f'F{stage-1}'],df[f'F{stage}']):
        draw_vector(ax, x, d0, 0, delta-d0, df.rent.max()-df.rent.min())

# PLOT 1

ax = axes[0]
line1, = ax.plot(df.sqfeet,df.rent,'o', linewidth=.8, markersize=4, label="$y$")
draw_stage_residual(ax, df, stage=1)
# fake a line to get smaller red dot
line2, = ax.plot(700,1000,linewidth=.8, c='r', markersize=4)
ax.plot([df.sqfeet.min()-10,df.sqfeet.max()+10], [f0,f0],
         linewidth=.8, linestyle='--', c='k')
ax.text(815, f0+10, r"$f_0({\bf x})$", fontsize=18)
ax.set_xlim(df.sqfeet.min()-10,df.sqfeet.max()+10)
ax.set_ylim(df.rent.min()-10, df.rent.max()+10)
ax.set_xlabel(r"SqFeet", fontsize=14)
ax.set_ylabel(r"Rent", fontsize=14)
ax.legend(handles=[line2], fontsize=15,
          loc='upper left', 
          labelspacing=.1,
          handletextpad=.2,
          handlelength=.7,
          frameon=True,
          labels=["$\Delta_1({\\bf x}; {\\bf w}_1=[20,100])$"])

# PLOT 2

ax = axes[1]
line1, = ax.plot(df.sqfeet,df.rent,'o', linewidth=.8, markersize=4, label="$y$")
draw_stage_residual(ax, df, stage=2)
# fake a line to get smaller red dot
line2, = ax.plot(700,1000,linewidth=.8, c='r', markersize=4)
ax.plot([df.sqfeet.min()-10,df.sqfeet.max()+10], [f0,f0],
         linewidth=.8, linestyle='--', c='k')
ax.text(815, f0+10, r"$f_0({\bf x})$", fontsize=18)
ax.set_xlim(df.sqfeet.min()-10,df.sqfeet.max()+10)
ax.set_ylim(df.rent.min()-10, df.rent.max()+10)
ax.set_xlabel(r"SqFeet", fontsize=14)
ax.legend(handles=[line2], fontsize=15,
          loc='upper left', 
          labelspacing=.1,
          handletextpad=.2,
          handlelength=.7,
          frameon=True,
          labels=["$\Delta_2({\\bf x}; {\\bf w}_2=[5,30])$"])

# PLOT 3

ax = axes[2]
line1, = ax.plot(df.sqfeet,df.rent,'o', linewidth=.8, markersize=4, label="$y$")
draw_stage_residual(ax, df, stage=3)
# fake a line to get smaller red dot
line2, = ax.plot(700,1000,linewidth=.8, c='r', markersize=4)
ax.plot([df.sqfeet.min()-10,df.sqfeet.max()+10], [f0,f0],
         linewidth=.8, linestyle='--', c='k')
ax.text(815, f0+10, r"$f_0({\bf x})$", fontsize=18)
ax.set_xlim(df.sqfeet.min()-10,df.sqfeet.max()+10)
ax.set_ylim(df.rent.min()-10, df.rent.max()+10)
ax.set_xlabel(r"SqFeet", fontsize=14)
ax.legend(handles=[line2], fontsize=15,
          loc='upper left', 
          labelspacing=.1,
          handletextpad=.2,
          handlelength=.7,
          frameon=True,
          labels=["$\Delta_3({\\bf x}; {\\bf w}_3=[5,20])$"])

plt.tight_layout()
plt.show()
</pyfig>

We've manually chosen these weights as nice round numbers so we can do the computations more easily.  In practice, the the listing algorithm must compute the optimal weights of a complicated expression you can find on page 6 of <a href="https://statweb.stanford.edu/~jhf/ftp/trebst.pdf">Friedman's paper</a>. As he points out, however, the direction vector case simplifies to the weight of a leaf being just the median of the true $y_i$ values for the observations in that leaf. Weight $w_l$ for leaf $l$ is $median(y_i - F_{m-1}(\vec x_i))$ for all $\vec x_i$ in leaf $l$. (See Algorithm 3 *LAD_TreeBoost* on page 7 of Friedman's paper.) In a sense, this weight computation is acting like the residual vector. Instead of a separate "weight" for each observation, however, we use a separate weight for all similar observations (those in the same leaf). This has the effect that the prediction for similar observations takes steps sized proportionally to the median distance from the weak model's prediction to the true target values. As the weak model predictions get better, the algorithm takes smaller steps to zero in on the best prediction.

<!-- composite model -->

<pyfig label=examples hide=true width="90%">
df = data()
eta = 1
mse,mae = boost(df, 'sqfeet', 'rent', splits, eta, stages)
df['deltas12'] = eta * df[['delta1','delta2']].sum(axis=1)
df['deltas123'] = eta * df[['delta1','delta2','delta3']].sum(axis=1)
df['deltas'] = eta * df[['wdelta1','wdelta2','wdelta3']].sum(axis=1) # sum deltas

# Iterate through F_'stage', finding split points in predicted rent
# and create coordinate list to draw lines
def get_combined_splits(stage):
    x_prev = np.min(df.sqfeet)
    y_prev = np.min(df[f'F{stage}'])
    X = df.sqfeet.values
    coords = []
    for i,y_hat in enumerate(df[f'F{stage}'].values):
        if y_hat!=y_prev:
            mid = (X[i]+X[i-1])/2
            coords.append((mid,y_prev))
            coords.append((mid,y_hat))
            coords.append((X[i],y_hat))
        else:
            coords.append((X[i],y_hat))
        y_prev = y_hat
    return coords

def plot_combined(ax, stage, coords):
    line1, = ax.plot(df.sqfeet,df.rent, 'o', label=r'$y$')
    prev = None
    for x,y in coords:
        if prev is not None:
            line2, = ax.plot([prev[0],x], [prev[1],y], linewidth=.8,
                             linestyle='--', c='k')
        prev = (x,y)

    ax.set_xlabel(r"SqFeet", fontsize=14)

    ax.set_yticks(np.arange(1150,1351,50))
    ax.set_xlim(df.sqfeet.min()-10,df.sqfeet.max()+10)
    labs = ['\Delta_1', '\Delta_2', '\Delta_3']
    if stage==1:
        label = r"$f_0 + \eta \Delta_1$"
    else:
        label=r"$f_0 + \eta ("+'+'.join(labs[:stage])+")$"
    ax.legend(handles=[line2], fontsize=16,
              loc='center left', 
              labelspacing=.1,
              handletextpad=.2,
              handlelength=.7,
              frameon=True,
             labels = [label])

fig, axes = plt.subplots(nrows=1, ncols=3, figsize=(11, 3), sharey=True)

axes[0].set_ylabel(r"Rent", fontsize=14)
coords = get_combined_splits(1)
plot_combined(axes[0], 1, coords)

coords = get_combined_splits(2)
plot_combined(axes[1], 2, coords)

coords = get_combined_splits(3)
plot_combined(axes[2], 3, coords)

plt.tight_layout()
plt.show()
</pyfig>

Now show addition of all terms, dsashed lines, visually.

$\hat y = f_0(\vec x) + \Delta_1(\vec x) + \Delta_2(\vec x) + ...  + \Delta_M(\vec x)$

That's easy enough, so what's the problem? How do we know this procedure is correct and terminates? Why do they call it gradient boosting?

we add together the results of multiple weak learners

How good is that model? To answer that, we need a loss or cost function, $L(y,\hat y)$, that computes the cost of predicting $\hat y$ instead of $y$.  The squared error, $L(y,\hat y) = (y-\hat y)^2$ is the most common, but sometimes we care more about the absolute difference, $L(y,\hat y) = |y-\hat y|$. The loss across all observations is just the sum (or the average if you want to divide by $N$) of all the individual observation losses:

\[
L(\vec y, X) = \sum_{i=1}^{N} L(y_i, F_M(\vec x_i))
\]

That gives this either $L(\vec y, X) = \sum_{i=1}^{N} (y_i - F_M(\vec x_i))^2$ or $L(\vec y, X) = \sum_{i=1}^{N} |y_i - F_M(\vec x_i)|$.

Have you ever wondered why this technique is called *gradient* boosting? We're boosting gradients because our weak models learn direction vectors, and the other common term for "direction vector" is, drumroll please, *gradient*.  that leads us to optimization via gradient descent.

\todo{ can mention momentum instead of or in addition to leaf weights.  see accelerated gradient boosting paper recently.}