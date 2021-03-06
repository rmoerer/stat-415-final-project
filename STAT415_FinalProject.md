A Bayesian Analysis of Receiving Performance in the 2020 NFL Season
Using a Hierarchical Model
================
Austin Berg and Ryan Moerer
3/12/2021

### Motivation

Football fans frequently make value judgments on their teams
performance. For example, when the Philadelphia Eagles lost what was
effectively their entire receiving core (specifically wide receivers
Nelson Agholor, Desean Jackson, and Alshon Jeffery, running back Miles
Sanders, and tight end Zach Ertz) to injury in 2019, heightened
attention was paid to the performance, or lack thereof, of the Eagles’
replacement receivers. It became a tricky situation for value judgments
- with little historical performance data for the new players, fans
didn’t know what to expect from the offense. *How would Eagles fans know
when to complain versus when to acclaim the team’s receivers?*

NFL scouts, performance analysts, and strategists might routinely ask a
similar question before a game or a season: **What can we expect,
production-wise, out of our receivers?** Factors critical to receiver
performance include the following:

-   Number of targets
-   Catch rate
-   Yards gained per catch

A simple model for estimating and predicting wide receiver performance
(such as the one described in this report) takes these three factors
into account. A more complex model may look at the effect factors such
as quarterback performance, defensive rating, and weather have on
expected performance. In this report, we will use the aforementioned
simple model in a Bayesian analysis of receiver performance, using data
from the 2020 NFL Season.

### Data Collection

The first step is to collect data. For this analysis, regular season
play-by-play performance data for all full/part-time starters at the
running back, tight end, and wide receiver position was collected via
the R package `nflfastR`. This data included information on whether or
not a pass was completed, the number of yards the play gained, and many
other factors as well. Receiving data aggregated at the season level was
also scraped from Pro Football Reference so information on the number of
games each player played could be gathered.

### Methodology

To estimate targets per game, catch rate, and yards per reception at
both the league-wide and individual player level, we used a hierarchical
Bayesian model with several assumptions. For a single player *j* we
assume that targets, catch rate, and yards gained per reception are
independent. We also assume

-   Conditional on *λ*<sub>*j*</sub>, the number of targets in a season
    *N* has a Poisson(*λ*<sub>*j*</sub>*G*) distribution where *G* is
    the number of games played
-   Conditional on *n*<sub>*j*</sub> and *p*<sub>*j*</sub>, the number
    of receptions in a season *Y* has a Binomial(*n*<sub>*j*</sub>,
    *p*<sub>*j*</sub>) distribution
-   Conditional on *μ*<sub>*j*</sub> and *σ*, the yards gained for a
    single reception *X* has a Normal(*μ*<sub>*j*</sub>, *σ*)
    distribution. We assume that the variance in yards gained is the
    same for every player

For the prior distribution we assume

-   *μ*<sub>*j*</sub> has a Normal(*μ*, *τ*) distribution
-   *p*<sub>*j*</sub> has a Beta(*μ*<sub>*p*</sub>*κ*<sub>*p*</sub> ,
    (1 − *μ*<sub>*p*</sub>)*κ*<sub>*p*</sub>) distribution
-   *λ*<sub>*j*</sub> has a
    Gamma(*μ*<sub>*λ*</sub><sup>2</sup>/(1/*τ*<sub>*λ*</sub><sup>2</sup>),
    *μ*<sub>*λ*</sub>/(1/*τ*<sub>*λ*</sub><sup>2</sup>)) distribution
-   *σ* has a Gamma(1, 1) distribution

For the hyperprior distribution we assume

-   *μ* has a Normal(10, 1) distribution
-   1/*τ*<sup>2</sup> has a Gamma(1, 0.1) distribution
-   *μ*<sub>*p*</sub> has a Beta(13, 7 distribution)
-   *κ*<sub>*p*</sub> has a Gamma(1, 0.1) distribution
-   *μ*<sub>*λ*</sub> has a Gamma(4, 1) distribution
-   1/*τ*<sub>*λ*</sub><sup>2</sup> has a Gamma(1, 0.1) distribution

The hyperpriors were largely chosen through our own knowledge of
football. For example, we thought the true catch rate would be around
65% so we chose a Beta distribution with a mean of 0.65. We also thought
the true number of targets per game would be around 3 so we chose a
Gamma distribution with a mode of 3. And finally we thought, on average,
about 10 yards would be gained per reception so we chose a Normal
distribution with a mean of 10. Of course, we were not too sure about
the true values for all three of these parameters, so we allowed for a
lot of uncertainty in our prior distributions. Furthermore, we
ultimately were not too sure about what *σ*, *τ*, *κ*<sub>*p*</sub>, and
*τ*<sub>*λ*</sub> should be so we chose largely uninformative priors and
hyperpriors for these parameters.

### The Model

The model was implemented in JAGS using the R package `rjags`. The full
model can be seen in the code below.

``` r
# read in data
pbp_data <- read.csv("./data/2020_nfl_receptions_pbp.csv")
season_data <- read.csv("./data/2020_nfl_receptions_season.csv")

# order pbp data by player id
pbp_data$receiver_id <- as.factor(pbp_data$receiver_id)
pbp_data <- pbp_data[order(pbp_data$receiver_id), ]

# order game data by player id
season_data$receiver_id <- as.factor(season_data$receiver_id)
season_data <- season_data[order(season_data$receiver_id), ]


# prepare data for JAGS
# first loop data
yds <- pbp_data$yards_gained[pbp_data$complete_pass == 1]
PlayerIndex <- as.numeric(factor(pbp_data$receiver_id[pbp_data$complete_pass == 1]))
# second loop data
rec <- season_data$Rec
tgt <- season_data$Tgt
G <- season_data$G
# number of players
J <- length(unique(PlayerIndex))


# model string
modelString <- "model{
  
  for (i in 1:length(yds)){
    yds[i] ~ dnorm(mu_j[PlayerIndex[i]], invsigma2)
  }
  
  for (j in 1:J){
    rec[j] ~ dbinom(p_j[j], tgt[j])
    tgt[j] ~ dpois(lambda_j[j] * G[j])
  }
  
  # priors
  for (j in 1:J){
    mu_j[j] ~ dnorm(mu, invtau2)
    p_j[j] ~ dbeta(mu_p * kappa_p, (1 - mu_p) * kappa_p)
    lambda_j[j] ~ dgamma(pow(mu_l, 2) / invtau2_l, mu_l / invtau2_l)
  }
  
  
  invsigma2 ~ dgamma(1, 1)
  sigma <- sqrt(pow(invsigma2, -1))
  
  
  # hyperpriors
  mu ~ dnorm(10, 1)
  invtau2 ~ dgamma(1, 1)
  tau <- sqrt(pow(invtau2, -1))
  mu_p ~ dbeta(13, 7)
  kappa_p ~ dgamma(1, 0.1)
  mu_l ~ dgamma(4, 1)
  invtau2_l ~ dgamma(1, 1)
  tau_l <- sqrt(pow(invtau2_l, -1))
}
"

# data list for JAGS
dataList <- list(yds = yds,
                 rec = rec,
                 tgt = tgt,
                 G = G,
                 PlayerIndex = PlayerIndex,
                 J = J)


# compile model
model <- jags.model(file = textConnection(modelString), 
                    data = dataList)
```

    ## Compiling model graph
    ##    Resolving undeclared variables
    ##    Allocating nodes
    ## Graph information:
    ##    Observed stochastic nodes: 11954
    ##    Unobserved stochastic nodes: 1012
    ##    Total graph size: 24941
    ## 
    ## Initializing model

``` r
# burn in
update(model, n.iter = 2000)

# generate mcmc samples
posterior_sample <- coda.samples(model,
                                 variable.names = c("mu", "sigma", "tau", "mu_p", "kappa_p", "mu_l", "tau_l", "mu_j", "p_j", "lambda_j"),
                                 n.iter = 10000,
                                 n.chains = 5)

# trace plot for population parameters
bayesplot::mcmc_combo(posterior_sample,
                      pars = c("mu", "sigma", "tau", "mu_p", "kappa_p", "mu_l", "tau_l"),
                      )
```

<img src="STAT415_FinalProject_files/figure-gfm/Model1: Normal Likelihood-1.png" style="display:block; margin:auto;" />

When we look at the trace plots and the posterior distributions,
everything seems to look pretty good. Almost all of our posterior
distributions for the parameters and hyperparameters, except for the
posterior distribution for *κ*<sub>*p*</sub>, are approximately normal.
However, before we interpret the results of our model lets make sure our
assumptions are correct and check to see how sensitive our results are
to changes in priors.

### Experimenting with Different Likelihoods

Those familiar with football might have noticed a problem with our
original assumptions. We assumed that yards gained on receptions were
normally distributed. But anyone who watches football knows that this is
most likely not the case. While most successful completions gain small
chunks of yardage down the field, the exciting part of football is the
large gains down the field that, while relatively rare, are certainly
more common than a normal distribution might assume. In fact, this trend
shows up quite clearly in our sample of 11,284 receptions from last
season. A histogram of yards gained on receptions from the 2020 NFL
season is shown below.

``` r
# histogram of observed data
hist(yds, freq=FALSE, breaks = 50)
```

<img src="STAT415_FinalProject_files/figure-gfm/Observed Data Viz-1.png" style="display:block; margin:auto;" />

The distribution is quite clearly right-skewed and not normally
distributed. Let’s use posterior predictive simulation to see what a
sample of 11,284 receptions would like under our original model. Density
lines for our model are shown in light blue in the graph below.

``` r
# histogram of observed data
hist(yds, freq=FALSE, breaks = 50)

# get possible values for mu and sigma
mu <- as.matrix(posterior_sample)[,"mu"]
sigma <- as.matrix(posterior_sample)[,"sigma"]

# posterior predictive simulation 
n = 11284
n_samples = 100
for (i in 1:n_samples){
  
  yds_sim = rnorm(n, mu[i], sigma[i])
  
  lines(density(yds_sim),
        col = rgb(135, 206, 235, max = 255, alpha = 25))
}
```

<img src="STAT415_FinalProject_files/figure-gfm/Posterior Sim 1-1.png" style="display:block; margin:auto;" />

Yikes, not a good fit at all. The skewness in the data is clearly not
accounted for by our model. Much *fewer* negative values were observed
than predicted by our model and *many more* large values were observed
than predicted by our model.

One method for adjusting our model to better reflect yards gained on
receptions is to assume that yards gained follows a distribution that
allows for a right skew. One simple implementation in JAGS would be to
assume that yards gained follows a Gamma distribution. But there is a
problem: A Gamma distribution only allows for positive values and, while
negative yardage receptions are rare, it is certainly possible to have a
reception that goes for negative yards. In order to account for this we
will add a constant of 20 yards to each observed value and assume that
for a single player *j*, conditional on *μ*<sub>*j*</sub> and *σ*, the
yards gained plus 20 for a single reception *X* has a
Gamma((*μ*<sub>*j*</sub> + 20)<sup>2</sup>/*σ*<sup>2</sup>,
(*μ*<sub>*j*</sub> + 20)/*σ*<sup>2</sup>) distribution. Our priors and
hyperpriors will remain the same.

``` r
# prepare data for JAGS
# first loop data
yds <- pbp_data$yards_gained[pbp_data$complete_pass == 1] + 20
PlayerIndex <- as.numeric(factor(pbp_data$receiver_id[pbp_data$complete_pass == 1]))
# second loop data
rec <- season_data$Rec
tgt <- season_data$Tgt
G <- season_data$G
# number of players
J <- length(unique(PlayerIndex))

# model string
modelString <- "model{
  
  for (i in 1:length(yds)){
    yds[i] ~ dgamma(pow(mu_j[PlayerIndex[i]] + 20, 2) / pow(sigma, 2), (mu_j[PlayerIndex[i]] + 20) / pow(sigma, 2))
  }
  
  for (j in 1:J){
    rec[j] ~ dbinom(p_j[j], tgt[j])
    tgt[j] ~ dpois(lambda_j[j] * G[j])
  }
  
  # priors
  for (j in 1:J){
    mu_j[j] ~ dnorm(mu, invtau2)
    p_j[j] ~ dbeta(mu_p * kappa_p, (1 - mu_p) * kappa_p)
    lambda_j[j] ~ dgamma(pow(mu_l, 2) / invtau2_l, mu_l / invtau2_l)
  }
  
  
  sigma <- sqrt(pow(invsigma2, -1))
  invsigma2 ~ dgamma(1, 1)
  
  
  
  # hyperpriors
  mu ~ dnorm(10, 1)
  invtau2 ~ dgamma(1, 0.1)
  tau <- sqrt(pow(invtau2, -1))
  mu_p ~ dbeta(13, 7)
  kappa_p ~ dgamma(1, 0.1)
  mu_l ~ dgamma(4, 1)
  invtau2_l ~ dgamma(1, 0.1)
  tau_l <- sqrt(pow(invtau2_l, -1))
}
"

# data list for JAGS
dataList <- list(yds = yds,
                 rec = rec,
                 tgt = tgt,
                 G = G,
                 PlayerIndex = PlayerIndex,
                 J = J)


# compile model
model <- jags.model(file = textConnection(modelString), 
                    data = dataList)
```

    ## Compiling model graph
    ##    Resolving undeclared variables
    ##    Allocating nodes
    ## Graph information:
    ##    Observed stochastic nodes: 11954
    ##    Unobserved stochastic nodes: 1012
    ##    Total graph size: 26283
    ## 
    ## Initializing model

``` r
# burn in
update(model, n.iter = 2000)

# generate mcmc samples
posterior_sample <- coda.samples(model,
                                 variable.names = c("mu", "sigma", "tau", "mu_p", "kappa_p", "mu_l", "tau_l", "mu_j", "p_j", "lambda_j"),
                                 n.iter = 10000,
                                 n.chains = 5)
```

``` r
# trace plots and posterior distributions
bayesplot::mcmc_combo(posterior_sample,
                      pars = c("mu", "sigma", "tau", "mu_p", "kappa_p", "mu_l", "tau_l"))
```

<img src="STAT415_FinalProject_files/figure-gfm/unnamed-chunk-1-1.png" style="display:block; margin:auto;" />

``` r
# generate histogram of observed data
yds <- yds - 20
hist(yds, freq=FALSE, breaks = 50)

# create posterior predictive samples
mu <- as.matrix(posterior_sample)[,"mu"]
sigma <- as.matrix(posterior_sample)[,"sigma"]
n = 11284
n_samples = 100

for (i in 1:n_samples){
  
  yds_sim = rgamma(n, (mu[i] + 20)^2 / sigma[i]^2, (mu[i] + 20) / sigma[i]^2) - 20
  
  lines(density(yds_sim),
        col = rgb(135, 206, 235, max = 255, alpha = 25))
}
```

<img src="STAT415_FinalProject_files/figure-gfm/unnamed-chunk-2-1.png" style="display:block; margin:auto;" />

The Gamma likelihood does not appear to do the best job either as large
gains still appear to happen more than the model predicts and gains in
yards between 0 and 10 happen much more than expected as well. However,
this model certainly fits the data much better than the previous model
as the new model allows for some skewness in the data, and negative
values are not predicted nearly as much as in the previous model.
Nonetheless, assuming that yards gained follows a Gamma distribution
seems to be a much more reasonable assumption than in our previous
model.

Now that we have tuned our model by adjusting our likelihood
assumptions, we will analyze the sensitivity our model has to changes in
priors.

### Sensitivity Analysis

One important part of interpreting the final results of our model is
determining how sensitive our results are to our priors. In order to
determine this sensitivity, we will choose wildly different hyperpriors
for *μ*, *μ*<sub>*λ*</sub> and *μ*<sub>*p*</sub> and see how the final
results compare to our original model.

We will assume

-   *μ* has a Normal(20, 0.5) distribution
-   *μ*<sub>*λ*</sub> has a Gamma(1, 10) distribution
-   *μ*<sub>*p*</sub> has a Beta(20, 40) distribution

``` r
# prepare data for JAGS
# first loop data
yds <- pbp_data$yards_gained[pbp_data$complete_pass == 1] + 20
PlayerIndex <- as.numeric(factor(pbp_data$receiver_id[pbp_data$complete_pass == 1]))
# second loop data
rec <- season_data$Rec
tgt <- season_data$Tgt
G <- season_data$G
# number of players
J <- length(unique(PlayerIndex))

# model string
modelString_sens <- "model{
  
  for (i in 1:length(yds)){
    yds[i] ~ dgamma(pow(mu_j[PlayerIndex[i]] + 20, 2) / pow(sigma, 2), (mu_j[PlayerIndex[i]] + 20) / pow(sigma, 2))
  }
  
  for (j in 1:J){
    rec[j] ~ dbinom(p_j[j], tgt[j])
    tgt[j] ~ dpois(lambda_j[j] * G[j])
  }
  
  # priors
  for (j in 1:J){
    mu_j[j] ~ dnorm(mu, invtau2)
    p_j[j] ~ dbeta(mu_p * kappa_p, (1 - mu_p) * kappa_p)
    lambda_j[j] ~ dgamma(pow(mu_l, 2) / invtau2_l, mu_l / invtau2_l)
  }
  
  
  invsigma2 ~ dgamma(1, 1)
  sigma <- sqrt(pow(invsigma2, -1))
  
  
  # hyperpriors
  mu ~ dnorm(20, 0.5)
  invtau2 ~ dgamma(1, 0.1)
  tau <- sqrt(pow(invtau2, -1))
  mu_p ~ dbeta(20, 40)
  kappa_p ~ dgamma(1, 0.1)
  mu_l ~ dgamma(1, 10)
  invtau2_l ~ dgamma(1, 0.1)
  tau_l <- sqrt(pow(invtau2_l, -1))
}
"

# data list for JAGS
dataList_sens <- list(yds = yds,
                 rec = rec,
                 tgt = tgt,
                 G = G,
                 PlayerIndex = PlayerIndex,
                 J = J)


# compile model
model_sens <- jags.model(file = textConnection(modelString_sens), 
                    data = dataList_sens)
```

    ## Compiling model graph
    ##    Resolving undeclared variables
    ##    Allocating nodes
    ## Graph information:
    ##    Observed stochastic nodes: 11954
    ##    Unobserved stochastic nodes: 1012
    ##    Total graph size: 26282
    ## 
    ## Initializing model

``` r
# burn in
update(model_sens, n.iter = 2000)

# generate mcmc samples
posterior_sample_sens <- coda.samples(model_sens,
                                 variable.names = c("mu", "sigma", "tau", "mu_p", "kappa_p", "mu_l", "tau_l", "mu_j", "p_j", "lambda_j"),
                                 n.iter = 10000,
                                 n.chains = 5)

# trace plot for population parameters
mcmc_combo(posterior_sample_sens[, c("mu", "mu_p", "mu_l")])
```

<img src="STAT415_FinalProject_files/figure-gfm/unnamed-chunk-3-1.png" style="display:block; margin:auto;" />

As we can see, despite choosing wildly different priors for *μ*,
*μ*<sub>*λ*</sub>, and *μ*<sub>*p*</sub>, the posterior distributions
for each parameter are quite similar to the posterior distributions in
our original model. This of course means that our results are not very
sensitive to changes in priors.

### Interpreting the Results

Now that we have a reasonable model, it is time to analyze posterior
results. Let’s first take another look at the posterior distributions
for mean targets per game, catch rate, and mean yards gained at the
league-wide level.

``` r
# posterior matrix
post_matrix <- as.matrix(posterior_sample)

# subset matrix into separate parameters
mu <- post_matrix[, "mu"]
mu_p <- post_matrix[, "mu_p"]
mu_l <- post_matrix[, "mu_l"]

# plot mu_lambda posterior
plotPost(mu_l)
```

<img src="STAT415_FinalProject_files/figure-gfm/mu_l Posterior-1.png" style="display:block; margin:auto;" />

As one can see, there is a 95% posterior probability that league-wide
mean targets per game is between 3.4 targets and 4.0 targets.

``` r
# plot mu_p posterior
plotPost(mu_p)
```

<img src="STAT415_FinalProject_files/figure-gfm/mu_p Posterior-1.png" style="display:block; margin:auto;" />

We also see that the true league-wide catch rate is between 0.67 and
0.69 with a posterior probability of 95%.

``` r
# plot mu posterior
plotPost(mu)
```

<img src="STAT415_FinalProject_files/figure-gfm/mu Posterior-1.png" style="display:block; margin:auto;" />

Furthermore, there is a posterior probability of 95% that the
league-wide mean yards gained per reception is between 10.7 and 11.4.
Since the posterior distributions for all three parameters are
approximately normal and are not correlated we can assume independence
and say that there is a 85.7% posterior probability that the true vales
for all three parameters are within the aforementioned intervals.

``` r
# create correlation matrix for parameters
cor(cbind(mu, mu_l, mu_p))
```

    ##                mu         mu_l         mu_p
    ## mu    1.000000000 -0.040904969  0.005574287
    ## mu_l -0.040904969  1.000000000 -0.006462245
    ## mu_p  0.005574287 -0.006462245  1.000000000

### Interpreting the Results for Individual Players

Now while it is important to have an understanding of the average
targets per game, the average catch rate, and the average yards gained
per reception at the league-wide level, a coach or a fan may be much
more concerned with the true values for these parameters at the player
level (take, for instance, Eagles fans in 2019). To get a sense of this
model’s value at the individual player level, let’s take a look at the
posterior distributions for yards per reception for all 2020 Eagles
receivers through the use of a caterplot.

``` r
# generate posterior summary stats
post_summary_stats <- summary(posterior_sample)$statistics

# create player summary for posterior distribution
player_summary <- cbind(season_data %>% mutate(Tgts_G = Tgt / G, Catch_Rt = Rec / Tgt, Yds_Rec = Yds/Rec),
                        lambda_j_post_mean =  post_summary_stats[str_detect(rownames(post_summary_stats), "lambda_j"),1],
                        lambda_j_post_sd = post_summary_stats[str_detect(rownames(post_summary_stats), "lambda_j"),2],
                        mu_j_post_mean = post_summary_stats[str_detect(rownames(post_summary_stats), "mu_j"), 1],
                        mu_j_post_sd = post_summary_stats[str_detect(rownames(post_summary_stats), "mu_j"), 2],
                        p_j_post_mean = post_summary_stats[str_detect(rownames(post_summary_stats), "p_j"), 1],
                        p_j_post_sd = post_summary_stats[str_detect(rownames(post_summary_stats), "p_j"), 2]
                        )

# break up mcmc samples into each parameter
player_targets <- post_matrix[, str_detect(string = colnames(post_matrix), "lambda_j")]
player_catchRt <- post_matrix[, str_detect(string = colnames(post_matrix), "p_j")]
player_yds <- post_matrix[, str_detect(string = colnames(post_matrix), "mu_j")]
sigma <- post_matrix[, "sigma"]
colnames(player_targets) <- season_data$Name
colnames(player_catchRt) <- season_data$Name
colnames(player_yds) <- season_data$Name

# get names of players on the Eagles
phi <- player_summary$Name[player_summary$Team=="PHI"]

# caterplot for yards
caterplot(player_yds[, phi])
```

<img src="STAT415_FinalProject_files/figure-gfm/unnamed-chunk-4-1.png" style="display:block; margin:auto;" />

The points represent the mean of the posterior distribution of yards
gained on receptions for each specific player, while the thick and thin
lines represent the 68% and 95% posterior credible intervals,
respectively. When looking at this plot, you can see the benefit of
estimating player performance using a hierarchical Bayesian model.

For example, take a look at the posterior distribution of mean yards
gained on receptions for Alshon Jeffery. Looking at the mean of this
posterior distribution, it appears that Jeffery is a pretty explosive
player. But his posterior distribution has a large degree of variance as
there is a 95% posterior probability that his true yards per reception
is between about 9.3 yards per reception and 17.8 yards per reception.
Why is that? Well, it turns out that Jeffery was injured for much of the
year and only had 6 total receptions on the season. Since there is very
little data on Alshon Jeffery, there is a large degree of variance in
our posterior belief of what his true mean yards gained on receptions
is.

The beauty of the Bayesian hierarchical model is that it essentially
accounts for that lack of information with a wide posterior distribution
that has a mean that is shrunk toward the league average. This idea of
shrinking the raw averages can be seen in the plot below.

``` r
# create shrinkage plot
player_summary %>%
  filter(Team == "PHI") %>%
  select(Name, Yds_Rec, mu_j_post_mean) %>%
  reshape2::melt() %>%
  mutate(label = ifelse(variable == "Yds_Rec", Name, "")) %>%
  ggplot(aes(x = variable, y = value, group = Name)) +
  geom_line() +
  ggrepel::geom_text_repel(aes(label = label), nudge_x = -0.04, hjust = 1, direction = "y") +
  labs(y = "Yards Per Reception") +
  theme_minimal() +
  theme(axis.title.x = element_blank(),
        axis.text.x = element_blank())
```

<img src="STAT415_FinalProject_files/figure-gfm/unnamed-chunk-5-1.png" style="display:block; margin:auto;" />

The lines for each particular player start at the players raw average
for yards gained on receptions as seen in data from 2020 and the lines
end at the average of each player’s posterior distribution for yards
gained per reception. As is shown, the model assigns much more realistic
mean yards gained on receptions to players with little data, such as
Alshon Jeffery. Similar caterplots can be generated for the other
parameters as well.

``` r
# targets per game caterplot
caterplot(player_targets[, phi])
```

<img src="STAT415_FinalProject_files/figure-gfm/unnamed-chunk-6-1.png" style="display:block; margin:auto;" />

``` r
# catch rate caterplot
caterplot(player_catchRt[, phi])
```

<img src="STAT415_FinalProject_files/figure-gfm/unnamed-chunk-6-2.png" style="display:block; margin:auto;" />

Of course, with estimates for three areas of football for 335 players,
it can be quite difficult to fully comprehend what the Bayesian
hierarchical model is doing. Now that we have some sense of how its
works for individual players, lets take a look at the posterior results
on a broader scale.

### Further Exploration of the Posterior Results

#### Posterior Mean Targets and Yards Gained per Catch versus 2020 Catches

``` r
ggplot(player_summary, aes(lambda_j_post_mean, mu_j_post_mean, color = Rec)) +
  geom_point(alpha=0.8, size = 2) +
  scale_color_gradient(low = "navy", high = "orangered")  +
  xlab('Posterior Mean Targets') +
  ylab('Posterior Mean Yards Gained per Catch') + 
  theme_minimal() +
  labs(color = '2020 Catches')
```

<img src="STAT415_FinalProject_files/figure-gfm/Posterior Results 1-1.png" style="display:block; margin:auto;" />

This plot gives you a sense of the posterior means for both targets per
game and yards gained per reception. It is also quite easy to see the
shrinkage of the model as players with small amounts of catches have
their posterior means shrunk towards the league average.

#### Posterior Catch Rate vs 2020 Catch Rate

``` r
ggplot(player_summary, aes(Catch_Rt, p_j_post_mean,  color = Rec)) +
  geom_point(alpha=0.4, size = 2) +
  scale_color_gradient(low = "navy", high = "orangered")  +
  xlab('2020 Catch Rate') +
  ylab('Posterior Catch Rate') + 
  theme_minimal() +
  labs(color = '2020 Catches') +
  geom_abline(x = player_summary$Catch_Rt, y = player_summary$Catch_Rt,
              linetype = 'dashed') +
  geom_hline(yintercept = mean(player_summary$Catch_Rt),
             linetype = 'dotted') +
  xlim(c(0,1)) + 
  annotate("text", x = 0.25, y = 0.69, label = paste("Mean Catch Rate"), color = 'black', fontface = 'bold') + 
  annotate("text", x = 0.625, y = 0.52, label = paste("Slope = 1"), color = 'black', fontface = 'bold', angle = 0)
```

<img src="STAT415_FinalProject_files/figure-gfm/Posterior Results 2-1.png" style="display:block; margin:auto;" />

As mentioned before, a key feature of this model is that it provides
performance estimates for players with little or no data on them based
on known relationships from the data. Thus, receivers with close to 0
catches in the 2020 season (shown by dark blue points) are assigned
realistic posterior catch rates despite bizarre catch rates in the data.
The players with lots of data from the 2020 NFL season (shown in bright
red) have posterior catch rates very close to what they recorded in the
2020 NFL season This attribute of the model makes sense and ultimately
makes this model an effective tool for estimation and prediction. Notice
that this essential function, of assigning realistic performance
parameters to receivers with little data, is applied not only to catch
rate but also to the other performance parameters.

#### Variability Analysis

``` r
ggplot(player_summary, aes(p_j_post_sd, mu_j_post_sd, color = Rec)) +
  geom_point(alpha=0.8, size = 2) +
  scale_color_gradient(low = "navy", high = "orangered")  +
  xlab('Standard Deviation in Catch Rate') +
  ylab('Standard Deviation in Yards Gained per Catch') + 
  theme_minimal() +
  labs(color = '2020 Catches')
```

<img src="STAT415_FinalProject_files/figure-gfm/Posterior Results 3-1.png" style="display:block; margin:auto;" />

This plot simply shows that posterior distributions for players with
lots of data from the 2020 NFL season has little variance compared to
players who have little data. This makes sense because we are much more
certain about performance parameters for players of which much is known
compared to players of which little is known.

#### Posterior Yards Gained versus Posterior Catch Rate

``` r
ggplot(player_summary, aes(p_j_post_mean, mu_j_post_mean, color = Rec)) +
  geom_point(alpha=0.8, size = 2) +
  scale_color_gradient(low = "navy", high = "orangered")  +
  xlab('Posterior Mean Catch Rate') +
  ylab('Posterior Mean Yards Gained per Catch') + 
  theme_minimal() +
  labs(color = '2020 Catches')
```

<img src="STAT415_FinalProject_files/figure-gfm/Posterior Results 4-1.png" style="display:block; margin:auto;" />

Interesting. When we originally specified the model, we assumed that
yards per reception and catch rate were independent. However, when we
look at the posterior means for each parameter there is quite clearly a
negative correlation between posterior mean catch rate and posterior
mean yards gained per catch at the player level. And as you start to
think about it, this ultimately makes sense as it is quite likely that
the average amount of yards gained on targets with a large probability
of being completed (e.g. short throws) is much smaller than the amount
of yards typically gained on targets with a low probability of
completion (e.g. long throws).

Now that we have a better understanding of our posterior results, lets
take a brief look at the model’s ability to make predictions.

### Posterior Predictive Distributions

As is always the case in the NFL, there were a number of over/under
[prop
bets](https://www.sportsbettingdime.com/news/nfl/wild-card-weekend-props-best-team-player-prop-bets-games-2021/)
available in the playoffs this season. For example, the over/under on
receiving yards for Tampa Bay Bucs wide receiver Chris Godwin was set at
70.5 for his wildcard weekend game against the Washington Football Team.
Lets take a look at if our model would have found any value.

``` r
# receiver and total
who <-  "C.Godwin"
OU <- 70.5

# subset posterior matrix into relevant parameters
player_targets <- post_matrix[, str_detect(string = colnames(post_matrix), "lambda_j")]
player_catchRt <- post_matrix[, str_detect(string = colnames(post_matrix), "p_j")]
player_yds <- post_matrix[, str_detect(string = colnames(post_matrix), "mu_j")]
sigma <- post_matrix[, "sigma"]
colnames(player_targets) <- season_data$Name
colnames(player_catchRt) <- season_data$Name
colnames(player_yds) <- season_data$Name

# num of sims
n = 10000
tgt <- rep(NA, n)
rec <- rep(NA, n)
yds <- rep(NA, n)

# generate posterior predictive distribution
for (i in 1:n){
  tgt[i] <- rpois(1, player_targets[i, who])
  rec[i] <- rbinom(1, size = tgt[i], prob = player_catchRt[i, who])
  # yds[i] <- sum(rnorm(n = rec[i], mean = player_Yds[i, who], sd = sigma[who]))
  yds[i] <- sum(rgamma(rec[i], (player_yds[i, who] + 20)^2 / sigma[i]^2, (player_yds[i, who] + 20) / sigma[i]^2) - 20)
}

# yardage df
total_yards <- data.frame(targets = tgt, 
                          yards = yds,
                          over = ifelse(yds > OU, "Yes", "No"))

# probability of hitting over
prob_over <- sum(yds > OU) / length(yds)

# visualize posterior predictive distribution
ggplot(total_yards, aes(x = yards, fill = over)) + 
  geom_histogram(alpha = 0.8, aes(fill=over)) + 
  geom_vline(xintercept = OU) +
  theme_minimal() +
  xlab('Total Receiving Yards')+
  ylab('Frequency') +
  ggtitle(who) + 
  annotate("text", x = 210, y = 500, label = paste("Fraction that Hit the Over\n=\n", prob_over), color = 'black', fontface = 'bold')+
  labs(fill = 'Did the Over Hit?')
```

<img src="STAT415_FinalProject_files/figure-gfm/Posterior Predictive Individuals-1.png" style="display:block; margin:auto;" />

As is shown, the posterior predictive distribution can be used to see if
your model gives an edge to the over or the under.

What if somebody wanted to bet on which group of receivers, say the
receivers on their fantasy team versus the receivers on their opponents
fantasy team, would have more total yards? This model can also
accommodate such an analysis (provided the receivers aren’t teammates).
To validate the model, uneven teams will be chosen in this analysis.
Specifically, Stefon Diggs, DeAndre Hopkins, and Travis Kelce on an
“All-Star” Team versus Antonio Gandy-Golden, Jake Butt, and Levine
Toilolo on a “Bench-Warmer” Team.

``` r
t1_tgt <- rep(NA, n)
t1_rec <- rep(NA, n)
t1_yds <- rep(NA, n)
t2_tgt <- rep(NA, n)
t2_rec <- rep(NA, n)
t2_yds <- rep(NA, n)

# Posterior preidctive distribution sim
# Bucs Receivers: Chris Godwin, Mike Evans, and Rob Gronkowski
# Chiefs Receivers: Travis Kelce, Tyreek Hill, and Sammy Watkins
for (i in 1:n){
  # team 1
  t1_tgt[i] <- rpois(1, player_targets[i, "S.Diggs"]) +
    rpois(1, player_targets[i, "D.Hopkins"]) +
    rpois(1, player_targets[i, "T.Kelce"])
  t1_rec[i] <- rbinom(1, size = tgt[i], prob = player_catchRt[i, "S.Diggs"]) +
    rbinom(1, size = tgt[i], prob = player_catchRt[i, "D.Hopkins"]) +
    rbinom(1, size = tgt[i], prob = player_catchRt[i, "T.Kelce"])
  t1_yds[i] <- sum(rgamma(rec[i], (player_yds[i, "S.Diggs"] + 20)^2 / sigma[i]^2, (player_yds[i, "S.Diggs"] + 20) / sigma[i]^2) - 20) +
    sum(rgamma(rec[i], (player_yds[i, "D.Hopkins"] + 20)^2 / sigma[i]^2, (player_yds[i, "D.Hopkins"] + 20) / sigma[i]^2) - 20) +
    sum(rgamma(rec[i], (player_yds[i, "T.Kelce"] + 20)^2 / sigma[i]^2, (player_yds[i, "T.Kelce"] + 20) / sigma[i]^2) - 20)
  
  # team 2
  t2_tgt[i] <- rpois(1, player_targets[i, "A.Gandy-Golden"]) +
    rpois(1, player_targets[i, "J.Butt"]) +
    rpois(1, player_targets[i, "L.Toilolo"])
  t2_rec[i] <- rbinom(1, size = tgt[i], prob = player_catchRt[i, "A.Gandy-Golden"]) +
    rbinom(1, size = tgt[i], prob = player_catchRt[i, "J.Butt"]) +
    rbinom(1, size = tgt[i], prob = player_catchRt[i, "L.Toilolo"])
  t2_yds[i] <- sum(rgamma(rec[i], (player_yds[i, "A.Gandy-Golden"] + 20)^2 / sigma[i]^2, (player_yds[i, "A.Gandy-Golden"] + 20) / sigma[i]^2) - 20) +
    sum(rgamma(rec[i], (player_yds[i, "J.Butt"] + 20)^2 / sigma[i]^2, (player_yds[i, "J.Butt"] + 20) / sigma[i]^2) - 20) +
    sum(rgamma(rec[i], (player_yds[i, "L.Toilolo"] + 20)^2 / sigma[i]^2, (player_yds[i, "L.Toilolo"] + 20) / sigma[i]^2) - 20)
}

# yardage df
total_yards <- data.frame(t1_yds = as.matrix(t1_yds), 
                          t2_yds = as.matrix(t2_yds))


# melt df
df <- reshape2::melt(total_yards)

# visualize posterior predictive dist for both groups
ggplot(df, aes(x = value, fill = variable)) + 
  geom_density(alpha = 0.5) + 
  theme_minimal() +
  xlab('Total Receiving Yards')+
  ylab('Density')
```

<img src="STAT415_FinalProject_files/figure-gfm/Posterior Predictive Teams-1.png" style="display:block; margin:auto;" />

``` r
# compute total yards
total_yards <- total_yards %>% mutate(t1_minus_t2 = t1_yds-t2_yds) %>% 
  mutate(positive = ifelse(t1_minus_t2>0, 1, 0))

# compute prob of team 1 having more yds
t1_more <- mean(total_yards$positive)

# visualize difference in yds
ggplot(total_yards, aes(x = t1_minus_t2, fill = factor(positive))) + 
  geom_histogram(alpha = 0.8, aes(fill=factor(positive))) + 
  geom_vline(xintercept = 0) +
  theme_minimal() +
  xlab('(All-Stars Yards) - (Bench-Warmer Yards)')+
  ylab('Frequency') +
  labs(fill = 'Winning Group') +
  scale_fill_discrete(labels = c("Bench-Warmers", "All-Stars"))  + 
  annotate("text", x = 220, y = 1000, label = paste("All-Stars Win Share \n=\n", t1_more), color = 'black', fontface = 'bold')
```

<img src="STAT415_FinalProject_files/figure-gfm/Posterior Predictive Teams-2.png" style="display:block; margin:auto;" />

As you can see, the posterior predictive model assigns significant
probability to the All-Star caliber Team 1 gaining more yards than
Bench-Warmer caliber Team 2.

### Conclusion

By using a hierarchical model, data from the 2020 NFL season, and
Bayesian statistics, we developed a simple tool for analyzing and
predicting NFL wide receiver performance. Despite the model’s
simplicity, it does seem generate reasonable estimates for player
performance. The posterior predictive distributions generated from the
model also allow us to visualize the outcomes of many simulated games
and identify inconsistencies between what our model predicts and betting
odds, for instance. With that said, this model can be easily further
developed by changing our assumptions and accounting for more factors in
our hierarchical model, such as receiver position, offensive team,
defensive team, weather, throwing quarterback, and many others.
