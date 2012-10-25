Quick trial SDP approaches with GP function
========================================================


```r
require(pdgControl)
```

```
## Loading required package: pdgControl
```

```r
require(ggplot2)
```

```
## Loading required package: ggplot2
```

```r
opts_knit$set(upload.fun = socialR::flickr.url)
```


### Beverton-Holt function

Simulate some training data under a stochastic growth function with standard parameterization,



```r
f <- BevHolt
p <- c(1.5, 0.05)
K <- (p[1] - 1)/p[2]
```



Noise function 

```r
z_g <- function(sigma_g) rlnorm(1, 0, sigma_g)  #1+(2*runif(1, 0,  1)-1)*sigma_g #
```



Parameter definitions


```r
x_grid = seq(0, 1.5 * K, length = 100)
T <- 40
sigma_g <- 0.1
x <- numeric(T)
x[1] <- 1
```


Simulation 


```r
for (t in 1:(T - 1)) x[t + 1] = z_g(sigma_g) * f(x[t], h = 0, p = p)
```




Predict the function over the target grid


```r
obs <- data.frame(x = x[1:(T - 1)], y = x[2:T])
X <- x_grid
library(nonparametricbayes)
gp <- gp_fit(obs, X, c(sigma_n = 1, l = 1))
```


Gaussian Process inference from this model.  True model shown in red.  


```r
df <- data.frame(x = X, y = gp$Ef, ymin = (gp$Ef - 2 * sqrt(abs(diag(gp$Cf)))), 
    ymax = (gp$Ef + 2 * sqrt(abs(diag(gp$Cf)))))
true <- data.frame(x = X, y = sapply(X, f, 0, p))
require(ggplot2)
ggplot(df) + geom_ribbon(aes(x, y, ymin = ymin, ymax = ymax), fill = "gray80") + 
    geom_line(aes(x, y)) + geom_point(data = obs, aes(x, y)) + geom_line(data = true, 
    aes(x, y), col = "red", lty = 2)
```

![plot of chunk unnamed-chunk-5](http://farm9.staticflickr.com/8473/8121208931_eb2e9cbdbe_o.png) 



## Stochastic Dynamic programming solution based on the posterior Gaussian process:

Define a transition matrix $F$ from the Gaussian process, giving the probability of going from state $x_t$ to $x_{t+1}$.
We already have the Gaussian process mean and variance predicted for each point $x$ on our grid, so this is simply:



```r
V <- sqrt(diag(gp$Cf))
F <- sapply(x_grid, function(x) dnorm(x, gp$Ef, V))
F <- t(apply(F, 1, function(x) x/sum(x)))  # normalize
h_grid <- x_grid
```


True $f(x)$


```r
mu <- sapply(x_grid, f, 0, p)
F_true <- sapply(x_grid, function(x) dnorm(x, mu, sigma_g))
F_true <- t(apply(F_true, 1, function(x) x/sum(x)))  # normalize
```


Somewhat silly way to adjust the matrix by harvest level:

```r
n <- length(h_grid)
per_harvest <- function(F, n) lapply(0:(n - 1), function(i) {
    d <- diag(1, nrow = (n - i))
    top <- matrix(0, nrow = i, ncol = n)
    top[, 1] <- 1
    side <- matrix(0, nrow = (n - i), ncol = i)
    out <- F %*% rbind(top, cbind(d, side))
    t(apply(out, 1, function(x) x/sum(x)))  # normalize
})

```


True F is okay but not quite right:


```r
matrices <- per_harvest(F_true, n)
opt <- find_dp_optim(matrices, x_grid, h_grid, 20, 0, profit, delta = 0.01)
```

```
## Error: object 'profit' not found
```

```r
plot(opt$D[, 1])
```

```
## Error: object 'opt' not found
```


whoops: gp inferred F isn't working:


```r
matrices <- per_harvest(F, n)
opt <- find_dp_optim(matrices, x_grid, h_grid, 20, 0, profit, delta = 0.01)
```

```
## Error: object 'profit' not found
```

```r
plot(opt$D[, 1])
```

```
## Error: object 'opt' not found
```



Old-school calculation method:



```r
profit <- profit_harvest(price = 1, c0 = 0, c1 = 0)
pdfn <- function(P, s) dlnorm(P, 0, s)
matrices <- determine_SDP_matrix(f, p, x_grid, h_grid, sigma_g, pdfn)
opt <- find_dp_optim(matrices, x_grid = x_grid, h_grid = h_grid, 
    OptTime = 20, xT = 0, profit = profit, delta = 0.05)
plot(opt$D[, 1])
```

![plot of chunk unnamed-chunk-11](http://farm9.staticflickr.com/8048/8121209259_1bae534bf4_o.png) 

