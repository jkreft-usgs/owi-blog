---
author: Laura DeCicco and Marcus Beck
date: 2016-06-16
slug: plotFlowConc
type: post
title: EGRET plotFlowConc using ggplot2
categories:
  - r
  - EGRET
---
Here is an example of using `ggplot2` with `EGRET` objects. Here, a function `plotFlowConc` was created:

``` r
library(EGRET)
library(ggplot2)
library(tidyr)
library(dplyr)


plotFlowConc <- function(eList, month = c(1:12), years = NULL, col_vec = c('red', 'green', 'blue'), alpha = 1, size = 1,  allflo = FALSE, ncol = NULL, grids = TRUE, scales = NULL, pretty = TRUE, use_bw = TRUE, fac_nms = NULL){
  
  localDaily <- getDaily(eList)
  localINFO <- getInfo(eList)
  localsurfaces <- getSurfaces(eList)
  
  # flow, date info for interpolation surface
  LogQ <- seq(localINFO$bottomLogQ, by=localINFO$stepLogQ, length.out=localINFO$nVectorLogQ)
  year <- seq(localINFO$bottomYear, by=localINFO$stepYear, length.out=localINFO$nVectorYear)
  jday <- 1 + round(365 * (year - floor(year)))
  surfyear <- floor(year)
  surfdts <- as.Date(paste(surfyear, jday, sep = '-'), format = '%Y-%j')
  surfmos <- as.numeric(format(surfdts, '%m'))
  surfday <- as.numeric(format(surfdts, '%d'))
   
  # interpolation surfaces
  yHat <- localsurfaces[,,1] 
  SE <- localsurfaces[,,2]
  ConcDay <- localsurfaces[,,3]

  # convert month vector to those present in data
  month <- month[month %in% surfmos]
  if(length(month) == 0) stop('No observable data for the chosen month')
  
  # salinity/flow grid values
  flo_grd <- LogQ

  # get the grid
  to_plo <- data.frame(date = surfdts, year = surfyear, month = surfmos, day = surfday, t(ConcDay))

  # axis labels
  ylabel <- localINFO$paramShortName
  xlabel <- 'LogQ'
  
  # reshape data frame, average by year, month for symmetry
  to_plo <- to_plo[to_plo$month %in% month, , drop = FALSE]
  names(to_plo)[grep('^X', names(to_plo))] <- paste('flo', flo_grd)
  to_plo <- tidyr::gather(to_plo, 'flo', 'res', 5:ncol(to_plo)) %>% 
    mutate(flo = as.numeric(gsub('^flo ', '', flo))) %>% 
    select(-date, -day) %>% 
    group_by(year, month, flo) %>% 
    summarize(
      res = mean(res, na.rm = TRUE)
    )
  
  # subset years to plot
  if(!is.null(years)){
    
    to_plo <- to_plo[to_plo$year %in% years, ]
    to_plo <- to_plo[to_plo$month %in% month, ]
        
    if(nrow(to_plo) == 0) stop('No data to plot for the date range')
  
  }

  # constrain plots to salinity/flow limits for the selected month
  if(!allflo){
    
    #min, max flow values to plot
    lim_vals<- group_by(data.frame(localDaily), Month) %>% 
      summarize(
        Low = quantile(LogQ, 0.05, na.rm = TRUE),
        High = quantile(LogQ, 0.95, na.rm = TRUE)
      )
  
    # month flo ranges for plot
    lim_vals <- lim_vals[lim_vals$Month %in% month, ]
    lim_vals <- rename(lim_vals, month = Month)
    
    # merge limits with months
    to_plo <- left_join(to_plo, lim_vals, by = 'month')
    to_plo <- to_plo[to_plo$month %in% month, ]
        
    # reduce data
    sel_vec <- with(to_plo, 
      flo >= Low &
      flo <= High
      )
    to_plo <- to_plo[sel_vec, !names(to_plo) %in% c('Low', 'High')]
    to_plo <- arrange(to_plo, year, month)
    
  }
  
  # months labels as text
  mo_lab <- data.frame(
    num = seq(1:12), 
    txt = c('January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December')
  )
  mo_lab <- mo_lab[mo_lab$num %in% month, ]
  to_plo$month <- factor(to_plo$month, levels =  mo_lab$num, labels = mo_lab$txt)
  
  # reassign facet names if fac_nms is provided
  if(!is.null(fac_nms)){
    
    if(length(fac_nms) != length(unique(to_plo$month))) stop('fac_nms must have same lengths as months')
  
    to_plo$month <- factor(to_plo$month, labels = fac_nms)
    
  }
  
  # make plot
  p <- ggplot(to_plo, aes(x = flo, y = res, group = year)) + 
    facet_wrap(~month, ncol = ncol, scales = scales)
  
  # return bare bones if FALSE
  if(!pretty) return(p + geom_line())
  
  # get colors
  cols <- col_vec
  
  # use bw theme
  if(use_bw) p <- p + theme_bw()
  
  p <- p + 
    geom_line(size = size, aes(colour = year), alpha = alpha) +
    scale_y_continuous(ylabel, expand = c(0, 0)) +
    scale_x_continuous(xlabel, expand = c(0, 0)) +
    theme(
      legend.position = 'top'
    ) +
    scale_colour_gradientn('Year', colours = cols) +
    guides(colour = guide_colourbar(barwidth = 10)) 
  
  # remove grid lines
  if(!grids) 
    p <- p + 
      theme(      
        panel.grid.major = element_blank(),
        panel.grid.minor = element_blank()
      )
  
  return(p)
    
}
```

Next, the function can be called with any `EGRET` object:

``` r
eList <-  Choptank_eList
plotFlowConc(eList)
```

<img src='/static/plotFlowConc/unnamed-chunk-2-1.png'/>