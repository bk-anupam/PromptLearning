### **Why Do Noisy Features Have Large Parameter Values?**  

When a machine learning model learns from data, it assigns **parameter values (weights)** to different features based on their influence on the target variable. **Noisy features**, which are **random and uninformative**, can still receive **large weights** under certain conditions, especially when regularization is absent. Here’s why:

### **1. Noisy Features Have No True Pattern, So the Model "Compensates"**
- A **useful feature** has a strong, consistent relationship with the target, so the model assigns a reasonable weight.  
- A **noisy feature**, on the other hand, has **random fluctuations**, meaning its effect on the target is **inconsistent** across samples.  
- Since the model still tries to fit the training data as accurately as possible, it **forces large weight values** onto noisy features to adjust for these random variations.

### **Example:**  
Imagine we are predicting house prices, and we mistakenly include a **random customer ID** as a feature.  
- This ID has **no real relationship** with the price, but because it is **random**, it might **accidentally** correlate with prices in the training set.  
- The model **memorizes** these random correlations and assigns **large weight values** to fit the noise perfectly.

### **2. Lack of Regularization Allows Large Weights**
Regularization techniques like **L1 (Lasso) and L2 (Ridge)** **penalize** large weights, preventing noise-driven weight inflation. However, **without regularization**:
- The model is free to assign **arbitrarily large** weights to noisy features.  
- This leads to **overfitting**, where the model performs well on training data but generalizes poorly to unseen data.

### **Mathematical Perspective (Effect of L2 Regularization)**
Without L2 regularization, the loss function (Mean Squared Error, for example) is:

$$
L = \sum (y_i - \hat{y}_i)^2
$$

With L2 regularization:

$$
L = \sum (y_i - \hat{y}_i)^2 + \lambda \sum w_j^2
$$

- The additional term $\lambda \sum w_j^2$ discourages large weight values.
- **Without it, weights can grow excessively, even for noisy features.**

### **3. High Model Complexity Enables Memorization of Noise**
- A simple linear model with few parameters cannot easily memorize noise.
- But **high-capacity models** (e.g., deep neural networks, large decision trees) have enough flexibility to **assign large weights to random features**.

### **Example in Neural Networks**
- In a **deep learning model**, each weight update during training minimizes the loss.
- If noise in the dataset **accidentally reduces the loss**, the optimizer assigns **higher weight values** to noisy features.
- Without regularization or dropout, noisy weights grow **disproportionately large**.


### **4. Noisy Features Have High Variance, Which Leads to Large Weights**
- **Good features** have stable, consistent effects on predictions → Model assigns moderate weight values.
- **Noisy features** have **high variance**, meaning their effects fluctuate randomly.
- The model **inflates weight values** to adjust for these fluctuations, mistakenly treating them as important.

### **Example of Variance Inflation**
Suppose a dataset contains a **spurious feature** (e.g., random stock market noise in predicting house prices).
- If some random fluctuations **align with** the training labels, the model gives them **high importance**.
- Large weights **amplify variance**, leading to overfitting.

### **5. What Does Model Sensitivity Mean?**
### **(A) Sensitivity to Input Variations**  
- A **sensitive model** reacts **strongly** to small changes in input data.
- A **robust model** maintains stable predictions despite minor input variations.

**Example:**  
- **High Sensitivity:** A neural network trained without regularization changes predictions drastically if input values slightly shift.  
- **Low Sensitivity (Robust Model):** A well-regularized model makes consistent predictions even when inputs contain small noise.

### **(B) Sensitivity to Noise & Overfitting**  
- A highly sensitive model will capture **noise**, leading to **overfitting**.
- Reducing sensitivity (via **regularization, dropout, or data augmentation**) improves **generalization**.

### **6. Why Regularization Prevents Noise from Becoming a Feature**  
Regularization techniques such as **L1, L2, and dropout** help prevent models from learning noise by:  

### **(A) L1 Regularization (Lasso) – Feature Selection**  
- Shrinks some feature weights to **zero**, effectively **removing unimportant features**.  
- Noise-related features will likely be **discarded**.  

### **(B) L2 Regularization (Ridge) – Penalizing Large Weights**  
- Reduces overfitting by **shrinking weights**, preventing the model from assigning **high importance to noisy features**.  

### **(C) Dropout – Random Neuron Deactivation**  
- Prevents memorization by forcing the network to learn **redundant representations** that generalize well.  

### **(D) Early Stopping – Preventing Overtraining**  
- Stops training **before** the model starts capturing noise.
- Helps maintain **generalization** by halting training when validation loss starts increasing.

### **7. Conclusion**
- **Noisy features receive large parameter values** because the model tries to compensate for their randomness.
- **Lack of regularization** allows noisy weights to grow excessively.
- **High model complexity** enables memorization of noise, further amplifying weight values.
- **High variance in noisy features** forces the model to assign large weights to minimize training loss.

---
### **Exponential Weighted Average (EWA)**

Exponential Weighted Average (EWA) is a technique used to compute a weighted average of a sequence of values, where more recent values receive higher weights. 

It is commonly used in:
- **Time series analysis** and **signal processing** to emphasize recent trends.
- **Smoothing noisy or fluctuating data**, such as stock prices or temperature readings.
- **Capturing underlying patterns** in sequential data.

**Mathematical Formula for EWA:**

The EWA at time step **t** is computed as:

$$
V_t = \beta \cdot V_{t-1} + (1 - \beta) \cdot X_t
$$

Where:
- $ X_t $ = Observed value at time step **t**.
- $ \beta $ = Smoothing factor (also called weight or decay factor), where $ 0 < \beta < 1 $.
- $ V_t $ = Exponential weighted average at time **t**.
- $ V_{t-1} $ = EWA from the previous time step.

### **Effect of β (Smoothing Factor)**
The choice of $ \beta $ affects how quickly the EWA adapts to changes in the data:
- **Higher $ \beta $ (e.g., 0.98)** → Gives more weight to past values, making the EWA smoother (averages over a larger number of previous values).
- **Lower $ \beta $ (e.g., 0.5)** → Reacts faster to new values but retains less memory of past values.

Approximate number of past entries considered in EWA:

$$
\frac{1}{1 - \beta}
$$

| **β Value** | **Averages Over (Approx.)** |
|------------|----------------------------|
| 0.9        | Last 10 entries             |
| 0.98       | Last 50 entries             |
| 0.5        | Last 2 entries              |


#### **Initialization Bias**
At the start, **$ V_0 $** is often initialized to **0**, leading to an early bias where initial values are underestimated.

To correct this, a **bias-corrected** version of EWA is used:

$$
\hat{V}_t = \frac{V_t}{1 - \beta^t}
$$

This correction helps make the early values more representative of the true moving average. As t increases, the bias correction term approaches 1, and the correction becomes negligible. This ensures bias correction is only significant for early values when bias is most pronounced and diminishes over time.

### **Key Takeaways**
- EWA is useful for tracking trends while reducing the effect of noise.
- A **higher $ \beta $** smooths data over a longer period, making it slower to respond to changes.
- A **lower $ \beta $** makes EWA more responsive to new values but less stable.
- Bias correction ensures early values are not skewed by initialization.

---
## Gradient Descent with Momentum

Gradient Descent with Momentum is an optimization algorithm used to find the minimum of a cost function. It enhances the standard Gradient Descent algorithm by addressing limitations such as slow convergence and oscillations around the minimum.

### Key Concept
The core idea is to compute exponentially weighted averages for gradients and then update weights using these new values. This approach accelerates gradient descent in relevant directions, leading to faster convergence and reduced oscillations.

### Pseudo Code
```python
# Initialize velocity terms
vdW = 0
vdb = 0

# On iteration t:
# Can be used with mini-batch or batch gradient descent
compute dw, db on current mini-batch

# Update velocity terms
vdW = beta * vdW + (1 - beta) * dW
vdb = beta * vdb + (1 - beta) * db

# Update weights
W = W - learning_rate * vdW
b = b - learning_rate * vdb
```

### Benefits of Momentum
1. **Faster Convergence:** Momentum speeds up gradient descent by helping the cost function reach its minimum more efficiently.
2. **Reduced Oscillations:** The algorithm smooths out oscillations, especially in regions where gradients vary significantly in different directions.
3. **Better Optimization in Deep Networks:** It is particularly beneficial for deep networks or large-scale datasets where standard gradient descent may struggle.

### Explanation of the Hyperparameter `beta`
- `beta` determines how much of the previous velocity is retained.
- A common choice is `beta = 0.9`, which works well in most cases.

### Mathematical Intuition
To understand why momentum is effective, consider a **ball rolling down a hill**:
- **Without Momentum:** The ball follows the slope exactly, which may cause it to oscillate in narrow valleys or get stuck in small pits.
- **With Momentum:** The accumulated velocity allows the ball to roll smoothly, overcoming minor obstacles and accelerating towards the global minimum.

By incorporating momentum, optimization algorithms become more stable and efficient, leading to better training performance in neural networks.

---
## **RMSProp (Root Mean Square Propagation)**

RMSProp is an optimization algorithm designed to improve gradient descent by **adapting the learning rate** for each parameter individually. It helps address some limitations of standard gradient descent and AdaGrad, particularly in deep neural network training.

### **Key Concepts**
#### **1. Adaptive Learning Rate**
- Unlike basic gradient descent with a **constant learning rate**, RMSProp **adapts the learning rate** for each parameter based on the history of squared gradients.
- The learning rate for a parameter is adjusted by dividing it by the square root of the **exponentially weighted moving average** of its squared gradients.
- **Effect:**  
  - Parameters with **larger gradients** get **smaller learning rates**.  
  - Parameters with **smaller gradients** get **larger learning rates**.  
  - This helps stabilize training and avoid large oscillations.

#### **2. Damping Term (Epsilon)**
- To prevent the learning rates from becoming **too small** over time, a small constant $ \epsilon $ is added inside the square root.
- This avoids division by zero and ensures numerical stability.

### **Mathematical Formulation**
For each weight $ W $ and bias $ b $, RMSProp updates them as follows:

1. Compute the gradients:  
   $$
   dW, db = \text{compute gradients on current mini-batch}
   $$

2. Compute the moving average of squared gradients:  
   $$
   s_{dW} = \beta \cdot s_{dW} + (1 - \beta) \cdot dW^2
   $$
   $$
   s_{db} = \beta \cdot s_{db} + (1 - \beta) \cdot db^2
   $$
   - The squared gradients are averaged using an **exponential moving average** with a decay rate **$ \beta $** (typically 0.9).

3. Update weights and biases:  
   $$
   W = W - \frac{\text{learning rate} \cdot dW}{\sqrt{s_{dW}} + \epsilon}
   $$
   $$
   b = b - \frac{\text{learning rate} \cdot db}{\sqrt{s_{db}} + \epsilon}
   $$

   - The denominator ensures that large gradients result in smaller updates, stabilizing learning.

### **Pseudo Code**
```python
# Initialize squared gradient accumulators
sdW = 0
sdb = 0

for each iteration t:
    # Compute gradients on the current mini-batch
    dW, db = compute_gradients()

    # Compute exponentially weighted moving average of squared gradients
    sdW = (beta * sdW) + (1 - beta) * dW**2  # Squaring is element-wise
    sdb = (beta * sdb) + (1 - beta) * db**2  # Squaring is element-wise

    # Update weights and biases
    W = W - learning_rate * dW / (sqrt(sdW) + epsilon)
    b = b - learning_rate * db / (sqrt(sdb) + epsilon)
```
---

## **Adam Optimization Algorithm**

Adam (Adaptive Moment Estimation) is an optimization algorithm that **combines the benefits of RMSProp and Momentum**. It is widely used in deep learning due to its adaptive learning rate and ability to handle sparse gradients effectively.

### **How Adam Works**
Adam builds upon two key ideas:

1. **Momentum (First Moment Estimation)**  
   - Adam keeps an exponentially weighted moving average of past gradients (like Momentum).
   - This helps accelerate convergence and smooth out updates.

2. **RMSProp (Second Moment Estimation)**  
   - Adam maintains an exponentially weighted moving average of past squared gradients (like RMSProp).
   - This allows it to adapt learning rates for each parameter.

3. **Bias Correction**  
   - Since the moving averages are initialized at zero, they are biased toward zero in early iterations.
   - Adam applies bias correction to compensate for this effect.

### **Mathematical Formulation**
For each weight $ W $ and bias $ b $, Adam updates them as follows:

1. **Compute Gradients**  
   $$
   dW, db = \text{compute gradients on current mini-batch}
   $$

2. **Update First Moment Estimate (Momentum Term)**  
   $$
   v_{dW} = \beta_1 \cdot v_{dW} + (1 - \beta_1) \cdot dW
   $$
   $$
   v_{db} = \beta_1 \cdot v_{db} + (1 - \beta_1) \cdot db
   $$

3. **Update Second Moment Estimate (RMSProp Term)**  
   $$
   s_{dW} = \beta_2 \cdot s_{dW} + (1 - \beta_2) \cdot dW^2
   $$
   $$
   s_{db} = \beta_2 \cdot s_{db} + (1 - \beta_2) \cdot db^2
   $$

4. **Bias Correction**  
   $$
   \hat{v}_{dW} = \frac{v_{dW}}{1 - \beta_1^t}
   $$
   $$
   \hat{v}_{db} = \frac{v_{db}}{1 - \beta_1^t}
   $$
   $$
   \hat{s}_{dW} = \frac{s_{dW}}{1 - \beta_2^t}
   $$
   $$
   \hat{s}_{db} = \frac{s_{db}}{1 - \beta_2^t}
   $$

5. **Update Parameters**  
   $$
   W = W - \frac{\alpha \cdot \hat{v}_{dW}}{\sqrt{\hat{s}_{dW}} + \epsilon}
   $$
   $$
   b = b - \frac{\alpha \cdot \hat{v}_{db}}{\sqrt{\hat{s}_{db}} + \epsilon}
   $$

### **Pseudo Code**
```python
# Initialize moment estimates and squared gradient accumulators
vdW, vdb = 0, 0
sdW, sdb = 0, 0

for each iteration t:
    # Compute gradients on the current mini-batch
    dW, db = compute_gradients()

    # Update biased first moment estimate (Momentum)
    vdW = (beta1 * vdW) + (1 - beta1) * dW
    vdb = (beta1 * vdb) + (1 - beta1) * db

    # Update biased second moment estimate (RMSProp)
    sdW = (beta2 * sdW) + (1 - beta2) * (dW**2)
    sdb = (beta2 * sdb) + (1 - beta2) * (db**2)

    # Compute bias-corrected estimates
    vdW_corrected = vdW / (1 - beta1**t)
    vdb_corrected = vdb / (1 - beta1**t)
    sdW_corrected = sdW / (1 - beta2**t)
    sdb_corrected = sdb / (1 - beta2**t)

    # Update parameters
    W = W - learning_rate * vdW_corrected / (sqrt(sdW_corrected) + epsilon)
    b = b - learning_rate * vdb_corrected / (sqrt(sdb_corrected) + epsilon)
```
---

## **Adam vs AdamW: Understanding Weight Decay and Regularization**

### **Weight Decay**

Weight decay is a regularization technique used to reduce the magnitudes of the network's weights during training. It helps prevent **overfitting** by discouraging overly complex models that fit the noise in the data.

### **Mathematics of Weight Decay**:
In optimizers like Adam and RMSProp, the learning rate can vary for each parameter. Weight decay works by subtracting a fraction of the weight values after computing the adaptive learning rate.

The standard weight decay formula is:
$$
w = w - \text{lr} \cdot w.grad - \text{lr} \cdot wd \cdot w
$$
Where:
- $ w $ represents the weight
- $ \text{lr} $ is the learning rate
- $ w.grad $ is the gradient of the weight
- $ wd $ is the weight decay factor

This equation can be rewritten as:
$$
w = w(1 - \text{lr} \cdot wd) - \text{lr} \cdot w.grad
$$
Here, the term $ (1 - \text{lr} \cdot wd) $ applies the weight decay to the weights, reducing them after each update.

### **L2 Regularization** (A Special Case of Weight Decay)

L2 regularization is a specific type of weight decay where a penalty term is added to the loss function. This penalty term is proportional to the squared magnitudes of the weights, helping the model avoid large weights that can lead to overfitting.

The L2 regularization equation can be rewritten as:
$$
w = w - \text{lr} \cdot (w.grad + wd \cdot w)
$$
This formulation holds true for simpler optimizers like **SGD**, where the learning rate is consistent across parameters.

In the context of loss, the regularization term is:
$$
\text{Loss with L2} = \text{Loss without regularization} + \lambda \cdot (\sum w^2)
$$
Where:
- $ \lambda $ is the regularization strength (also known as the weight decay factor).

### **Adam's Original Weight Decay Implementation**
In frameworks like **PyTorch**, **TensorFlow**, and **Keras**, the original **Adam optimizer** applies weight decay by adding a fraction of the weights to the gradients before updating the parameters. This method, however, is equivalent to **L2 regularization** and is **inconsistent** with the definition of weight decay, which involves subtracting the weight fraction directly from the parameters.

While this distinction may not matter for simpler optimizers like **SGD**, it becomes significant when using **adaptive optimizers** like **Adam**, which have different learning rates for each parameter.

## **Adam Optimization Algorithm - Mathematical Formulation**

Adam is an optimizer that combines the benefits of **momentum** (via first moment estimation) and **adaptive learning rates** (via second moment estimation). Below are the key steps involved in the Adam optimization algorithm:

### 1. **Gradient Calculation**:
$$
g_t = \nabla_{\theta_t} f(\theta_t) + \lambda \theta_t
$$
Where:
- $ g_t $ is the gradient of the loss function with respect to parameter $ \theta_t $.
- $ \lambda $ is the regularization term, which helps control the magnitude of the weights.

Adam implementation in all libraries we have looked at use this form. (In practice, it is nearly always implemented by adding wd*w to the gradients, rather than actually changing the loss function: we don’t want to add more computations by modifying the loss when there is an easier way.)

### 2. **First Moment Estimate (Momentum Term)**:
$$
m_t = \beta_1 m_{t-1} + (1 - \beta_1) g_t
$$
Here, $ m_t $ is the moving average of past gradients, and $ \beta_1 $ is the decay rate for this moving average.

### 3. **Second Moment Estimate (RMSProp Term)**:
$$
v_t = \beta_2 v_{t-1} + (1 - \beta_2) g_t^2
$$
Here, $ v_t $ tracks the exponentially decaying average of the squared gradients, and $ \beta_2 $ is the decay rate for this moving average.

### 4. **Bias Correction**:
To correct the bias introduced by the initial zero values of $ m_t $ and $ v_t $, we use the following bias-corrected estimates:
$$
\hat{m}_t = \frac{m_t}{1 - \beta_1^t}
$$
$$
\hat{v}_t = \frac{v_t}{1 - \beta_2^t}
$$

### 5. **Parameter Update Rule**:
The parameter update is performed as:
$$
\theta_{t+1} = \theta_t - \alpha \frac{\hat{m}_t}{\sqrt{\hat{v}_t} + \epsilon}
$$
Where:
- $ \theta_t $ is the model parameter at time step $ t $
- $ \hat{m}_t $ and $ \hat{v}_t $ are the bias-corrected moment estimates
- $ \alpha $ is the learning rate
- $ \epsilon $ is a small constant to avoid division by zero


### **Key Notations**:
- $ \theta_t $: Model parameter at time step $ t $
- $ g_t $: Gradient at time step $ t $
- $ m_t $: First moment estimate (Momentum term)
- $ v_t $: Second moment estimate (RMSProp term)
- $ \beta_1, \beta_2 $: Exponential decay rates for moment estimates
- $ \alpha $: Learning rate
- $ \epsilon $: Smoothing term to prevent division by zero


## **AdamW: A Modified Version of Adam**

**AdamW** is a modification of the Adam optimizer that fixes the issue with weight decay. In AdamW:
- **Weight decay is decoupled** from the gradient update.
- The weight decay is directly applied by **subtracting** a fraction of the weights after computing the adaptive learning rate, following the **original definition** of weight decay.

This decoupling of weight decay and adaptive learning helps improve the performance and convergence of the model, particularly in **large and complex models**.

### **AdamW Implementation**:
In AdamW, the weight update rule becomes:
$$
\theta_{t+1} = \theta_t - \alpha \frac{\hat{m}_t}{\sqrt{\hat{v}_t} + \epsilon} - \alpha \cdot wd \cdot \theta_t
$$
Where $ wd $ represents the weight decay term applied directly to the weights.

By ensuring proper implementation of weight decay, AdamW helps achieve better generalization, especially in large-scale deep learning models.