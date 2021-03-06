# Park_capstone
## 1) Backgroudn and significance
Bacterial persistence refers to the survival of a fraction of the bacterial population under the treatment of antibiotics. This can be explained by bacterial cells in the "persister" state, which do not divide and are refractory to killing by antibiotics. However, these persister cells spontaneously resume to grow and susceptible to antibiotics again. As the existence of persister cells can reduce the effectiveness of treatment, understanding how this persister population will resume growing under different stresses is important.



## 2) Remaining questions
Even though recent studies suggested a number of the mechanism of persistence under various stresses, it is not known whether different antibiotic stress affects the rate of which persister cells resume to grow (rejuvenation rate, $k$) of persister cells differently.  

## 3) Prediction
If the persister population is treated with different antibiotics, the rejuvenation rate of cells in each population will vary, which will be observed as a different rate of population decay caused by the antibiotic killing of rejuvenated cells. 

## 4) Independent and dependent variables
The dependent variable will be the rejuvenation rate, which is assumed to be equivalent to the rate of population decay under antibiotic treatment. This rate can be estimated from the changes in the number of viable cells in a persister population during antibiotic treatment as the number of viable cells will be decreased, as described in the equation below. 

${N_t=N_0{\times}e^{-kt}}$, 
where $N_t$ and $N_0$ are the number of viable cells at time t and 0, respectively, and $k$ is the rejuvenation rate. Even though the number of viable cells across time is a discrete measured variable, the rate of population decay (or the rejuvenation rate) estimated from the number of cells will be a continuous variable. 

The independent variable of the experiment will be the type of antibiotics treated to the persister population, which is discrete, factorial variables. 


## 5) Statistical hypothesis

The null hypothesis is that the variance explained by the model, or here, the type of antibiotic treatment, is less than or equal to the residual variance. The alternative hypothesis is that the variance associated with the model is larger than the residual variance. 


## 6) Statistical test 
One-way ANOVA for completely randomized measures will be performed for comparing the estimated $k$. As $k$ of the bacterial population will be estimated for each plate, it is completely randomly assigned to different antibiotics and thus not intrinsically-linked. Thus, one-way ANOVA for CR is suitable for the statistical test as there is one factorial independent variable and continuous, completely randomized dependent variable. 

The pairwise t-test can be further performed if we need to understand which antibiotics are different from which in terms of its effect on the rejuvenation rate, but as we are only interested in whether the impact of different antibiotics on the rate is different or not, a pairwise t-test is not needed. 

For estimation of the $k$ from the number of viable cells will be done by nonlinear regression of independent replicates as each population will be independent of each other, and the number of cells is expected to decrease in an exponential manner. 

## 7) Experimental procedures and design
Each plate with a bacterial population will be used as an experimental unit and will be randomly assigned to different antibiotic treatments. Each plate, thus, will be considered as an independent replicate. There will be four different antibiotics, and our primary endpoint is to test whether the rejuvenation rate under different antibiotic treatments is different or not. 

The type I and II error threshold will be 5% and 10%, respectively, and the number of replicates for each antibiotic will be determined based on a power of 90% as the type II error tolerance level will be 10%. 

## 8) Simulation for the expected results
```{r, echo = FALSE, warning=FALSE}
library(nlfitr)
library(ggplot2)
library(viridis)
library(tidyverse)
library(cowplot)
library(dplyr)
library(broom)
library(ez)
```

```{r}
# datamaker function
persistence_datamaker <- function (reps, timepoint, k, sd, ylo, yhi)
{
  data <- data.frame()
  for (i in 1:length(k))
  {
    tmp <- simdecay1(x=timepoint, k=k[i], ylo=ylo, yhi=yhi, sd=sd, reps=reps)
    tmp$data$k  <- k[i]
    tmp$data$id <- paste0(letters[i],rep (1:reps, each = length(timepoint)))
    
    data <- bind_rows(data, tmp$data)
    
  }
  
  return (data)
}

```



```{r}
# parameters
timepoint = c(0,3,8, 11, 13, 16, 18, 20, 22, 24, 30, 35 )
reps = 7
k <- c(0.03, 0.04, 0.05, 0.06)
sd = 5000
ylo = 0; yhi = 1e+5

# simulate
simdata = persistence_datamaker(reps = reps, timepoint = timepoint, k = k, sd = sd, ylo = ylo, yhi = yhi) 
head(simdata)
```

```{r}
# plot the simulation
ggplot(simdata, aes(x, y, group=id, color = as.factor(k)))+
  geom_line()+
  geom_point()+
  scale_color_viridis(discrete=T, name = "Antibiotics", labels = c("A", "B", "C", "D"))+
  labs(y = expression("The number of viable cells ("~N[CFU]~")"), 
       x = "Time (min)")+
  theme_bw()+
  ggtitle ("Simulation for the decay of persister population under different antibiotics")

```


```{r}
nloutput <- sapply(foo <- unique(simdata$id),
                   function (foo) fitdecay1(x, y, data=subset(simdata, id==foo),
                                            k=0.3, ylo=ylo, yhi=yhi, weigh=F),
                   simplify = F)
onephaseFits <- bind_rows(lapply(nloutput, tidy)) %>% 
  add_column(id=rep(unique(simdata$id), each=3), .before=T)

estimated_K <- onephaseFits %>%
  select(id, term, estimate) %>%
  filter(term == "k") %>%
  mutate(antibiotics = as.factor(rep(letters[1:length(k)], each = reps)), 
         wid = as.factor(1:(length(k) * reps)))

summary_k <- estimated_K %>% group_by(antibiotics) %>%
  summarise(mean_k=mean(estimate), 
            sd_k=sd(estimate))


summary_ylo <- onephaseFits %>%
  select(id, term, estimate) %>%
  filter(term == "ylo") %>%
  mutate(antibiotics = as.factor(rep(letters[1:length(k)], each = reps)), 
         wid = as.factor(1:(length(k) * reps))) %>%
  group_by(antibiotics) %>%
  summarise(mean_ylo=mean(estimate), sd_ylo=sd(estimate))


summary_yhi <- onephaseFits %>%
  select(id, term, estimate) %>%
  filter(term == "yhi") %>%
  mutate(antibiotics = as.factor(rep(letters[1:length(k)], each = reps)), 
         wid = as.factor(1:(length(k) * reps))) %>%
  group_by(antibiotics) %>%
  summarise(mean_yhi=mean(estimate), sd_yhi=sd(estimate))


left_join(left_join(summary_k, summary_ylo), summary_yhi )
```





```{r}

p1 <- ggplot(simdata, aes(x, y, color=as.factor(k), group=as.factor(k))) + 
  stat_summary(fun.data="mean_sdl",
               fun.args=list(mult=1))+
  scale_color_viridis(discrete=T, name = "Antibiotics", labels = c("A", "B", "C", "D"))+
  labs(y = expression("The number of viable cells ("~N[CFU]~")"), 
       x = "Time (min)")+
   theme_bw()


#  stat_function(fun= function(.x) 852*exp(.x*0.106))+
#  stat_function(fun= function(.x) 816*exp(.x*0.156))+

  
  #(yhi-ylo)*exp(-1*k*x) + ylo
  
  
p2 <- ggplot(estimated_K, aes(antibiotics, estimate, color=antibiotics))+
  geom_jitter(width=0.1, size=4)+
  stat_summary(fun.data=mean_sdl,fun.args=list(mult=1), geom="crossbar")+
  scale_color_viridis(discrete=T, name = "Antibiotics", labels = c("A", "B", "C", "D"))+
  labs(y = "Rejuvenation rate, k, min^-1", 
       x = "Antibiotics")+
   theme_bw()

prow <- plot_grid(
  p1+theme(legend.position = "none"),
  p2+theme(legend.position = "none"),
  labels="AUTO"
)

legend <- get_legend(p1+theme(legend.box.margin=margin(0,0,0,12)))

plot_grid(prow, legend, rel_widths=c(2, 0.4))



```
As shown in the figure above, the variation in the number of viable cells are expected and thus, the variation in the estimated k is also expected. 


# 9) Monte Carlo power analysis


```{r, warning = FALSE}
set.seed(15)
sims = 100
# Run the power simulator
rep = 7
pval <- replicate(
  sims, {
    
    sample.df <- persistence_datamaker(reps = reps, timepoint = timepoint, 
                                       k = k, sd = sd, yhi = yhi, ylo = ylo)
    #sample.df[2,2]
    # here, put data munging part
    
    
    nloutput <- sapply(foo <- unique(sample.df$id),
                       function (foo) fitdecay1(x, y, data=subset(sample.df, id==foo),
                                                k=0.3, ylo=ylo, yhi=yhi, weigh=F),
                       simplify = F)
    onephaseFits <- bind_rows(lapply(nloutput, tidy)) %>% 
      add_column(id=rep(unique(sample.df$id), each=3), .before=T)
    
    estimated_K <- onephaseFits %>%
      select(id, term, estimate) %>%
      filter(term == "k") %>%
      mutate(antibiotics = as.factor(rep(letters[1:length(k)], each = reps)), 
             wid = as.factor(1:(length(k) * reps)))
    
    sim.ezaov <- ezANOVA(
      data = estimated_K, 
      wid = wid,
      dv = estimate,
      between = antibiotics,
      type = 2
    )
    
    #sim.ezaov$ANOVA
    pval <- sim.ezaov$ANOVA[1,5];pval
    
  }
)

pwr.pct <- sum(pval<0.05)/sims*100
paste(pwr.pct, sep="", "% power with 7 replicates.")





```




