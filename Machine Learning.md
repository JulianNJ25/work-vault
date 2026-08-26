
# Supervised Learning with scikit-learn

## Binary classification

**Two types of supervised learning**:
- classification
- regression

Binary classification is used to predict a target variable that has only two labels (ex. 0 or 1)

### Classifying labels of unseen data

1. Build a model
2. Model lerns from label data passed to it
3.  Pass unlabeled data to the model as input
4. Model predicts the labels of the unseen data

> Labeled data = training data

###  k-Nearest Neighbors
algorithm for classification problems

- Predicts the label of a data point by
	- Looking at the `k` closest labeled data points
	- Taking a majority vote (maked the prediction based on the amount of closest neighbors)

ex .
![[Pasted image 20260825132611.png]]
> In the image the closests dots to the black dot are **RED** so the machine will determine that the dot is clasified in the red group - `k` = 2

ex 2.
![[Pasted image 20260825133054.png]]
> Here if `k` = 5 we would classify the dot as blue, since there is a mayority

