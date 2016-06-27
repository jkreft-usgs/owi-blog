---
author: David Watkins
date: 2016-06-23
slug: stats-service-map
status: post
title: Using the dataRetrieval Stats Service
categories:
  - r
  - draft
---
This script utilizes the new dataRetrieval package access to the [USGS Statistics Web Service](http://waterservices.usgs.gov/rest/Statistics-Service.html). We will be pulling near real-time (within the hour) data using the instantaneous value service in readNWIS data, and using the stats service data to put it in context of the site's history. At the time of this writing (June 23rd) a storm system had just passed through the OH-WV-PA tri-state area, and the map below shows the increased stream discharges. The script is setup to pull the most recent data, so if you run this code now your map will reflect the current conditions.

I. Get the data
---------------

There are three separate dataRetrieval calls here — to retrieve the relevant site IDs, retrieve the statistics for each site, and to retrieve the current data. These three data frames are joined by site number via [dplyr's](https://cran.rstudio.com/web/packages/dplyr/vignettes/introduction.html) left\_join function. Then we add a column to the final data frame to hold the color value for each station.

``` r
#example stats service map, comparing real-time current discharge to history for each site
#reusable for other state(s)
#David Watkins June 2016

library(dataRetrieval)
library(maps)
library(dplyr)
library(lubridate)

#pick state(s)
states <- c("Ohio","West Virginia","Pennsylvania")
tz <- "America/New_York"

#find sites
sites=data.frame()
for(st in states){
  stateCd <- stateCdLookup(st)
  stateSites <- whatNWISsites(stateCd=stateCd, parameterCd = "00060", hasDataTypeCd = "iv",siteType="st")
  stateSites$state <- st
  sites <- rbind(sites,stateSites)
}

#retrieve stats data, dealing with 10 site limit to stat service requests
reqBks <- seq(1,nrow(sites),by=10)
statData <- data.frame()
for(i in reqBks) {
  getSites <- sites$site_no[i:(i+9)]
  currentSites <- readNWISstat(siteNumbers = getSites,parameterCd = "00060", 
                               statReportType = "daily",statType=c("p10","p25","p50","p75","p90","mean"))
  statData <- rbind(statData,currentSites)
}

today <- Sys.time()
statDataToday<-statData[statData$month_nu==month(today) & statData$day_nu==day(today),]

#retrieve current data, pull out most recent data point
#looped to allow larger queries
ivData <- data.frame()
for(st in states){
  stIV <- renameNWISColumns(readNWISdata(service="iv",parameterCd="00060",sites=sites$site_no[sites$state==st],
                                           startDate=date(today),tz=tz))
  stIV <- subset(stIV,select=c(agency_cd,site_no,dateTime,Flow_Inst,tz_cd))
  ivData <- rbind(ivData,stIV)
}

mostRecent <- data.frame()
for(site in unique(ivData$site_no)){
  siteData <- ivData[ivData$site_no==site,]
  mostRecent <- rbind(mostRecent,tail(siteData,1))
}

firstJoin <- left_join(mostRecent,sites, by="site_no")
finalJoin <- left_join(statDataToday,firstJoin,by="site_no")
finalJoin <- finalJoin[!is.na(finalJoin$Flow_Inst),] #remove sites without current data

#classify current discharge values
finalJoin$class <- NA
finalJoin$class <- ifelse(is.na(finalJoin$p25), ifelse(finalJoin$Flow_Inst > finalJoin$p50_va, "cyan","yellow"),
                          ifelse(finalJoin$Flow_Inst < finalJoin$p25_va, "red2",
                                 ifelse(finalJoin$Flow_Inst > finalJoin$p75_va, "navy","green4")))
```

II. Make the plot
-----------------

The base map consists of two plots. The first makes the county lines with a gray background, and the second overlays the heavier state lines. After that we add the points for each stream gage, colored by the column we added to finalJoin. In the finishing details, grconvertXY is a handy function that converts your inputs from a normalized (0-1) coordinate system to the actual map coordinates, which allows the legend and scale to stay in the same relative location on different maps.

``` r
map('county',regions=states,fill=TRUE,col="gray87",lwd=0.5)
map('state',regions=states,fill=FALSE,lwd=2,add=TRUE)
points(finalJoin$dec_long_va,finalJoin$dec_lat_va,col=finalJoin$class,cex=1.2,pch=19)
box(lwd=2)
title(paste("Instantaneous discharge value (Q) percentile rank\n",today),line=1)
par(mar=c(5.1, 4.1, 4.1, 6), xpd=TRUE)
legend("bottomright",inset=c(0.1,-.2),legend=c("Q > P50*","Q < P50*","Q < P25","P25 < Q < P75","Q > P75"),
       pch=22,cex=1.2,pt.bg=c("cyan","yellow","red2","green4","navy"),ncol=2,title="Legend")
map.scale(ratio=FALSE,grconvertX(0.08,"npc"), grconvertY(0.08, "npc"))
text("*Other percentiles not available for these sites",font=3,x=grconvertX(0.7,"npc"),y=grconvertY(-0.055, "npc"))
```

<img src='/static/stats-service-map/stats-service-map.png'/>