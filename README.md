Indigenous-led Caribou Conservation
================
Clayton T. Lamb, Liber Ero Postdoctoral Fellow, University of British
Columbia
13 November, 2020

### Load Data

``` r
library(here)
library(ggrepel)
library(sf)
library(ggmap)
library(raster)
library(mapview)
library(RColorBrewer)
library(lwgeom)
library(ggpubr)
library(fasterize)
library(ggspatial)
library(paletteer)
library(piggyback)
library(usethis)
library(knitr)
library(tidyverse)


#Run to get local data up to git. Too large to upload naturally to git (files>100mb)

##Save output, using piggyback because too large for github, then delete file so it doesn't upload
# ##need a github token
# ##get yours here:
# browse_github_pat()
# github_token()
# ##Add to your r environ 
#  
# ##bring up R envir using this command below
# edit_r_environ()
# 
# #add this line
# GITHUB_TOKEN="***ADD TOKEN HERE***"
#
#
###OR for 1 time use add it here and run
#Sys.setenv(GITHUB_TOKEN="***ADD TOKEN HERE***")
# 
# piggyback::pb_upload("/Users/clayton.lamb/Google Drive/Documents/University/PDF/analyses/KZstory/spatial_data.zip",
#           tag = "spatial_data",
#           overwrite=TRUE)


##Download data here
# piggyback::pb_download(file="spatial_data.zip", 
#             repo = "ctlamb/Indigenous-Led-Caribou-Conservation-KZ",
#             dest = here::here(),
#             overwrite = TRUE,
#             show_progress = TRUE)

##Unzip and you will have the folder called "data" with all the data required to run this
```

### Set ggplot themes

``` r
##ggplot color/fillscales
scale_fill_Publication <- function(...){
  library(scales)
  discrete_scale("fill","Publication",manual_pal(values = c("#386cb0","#fdb462","#7fc97f","#ef3b2c","#662506","#a6cee3","#fb9a99","#984ea3","#ffff33")), ...)
}
scale_colour_Publication <- function(...){
  library(scales)
  discrete_scale("colour","Publication",manual_pal(values = c("#386cb0","#fdb462","#7fc97f","#ef3b2c","#662506","#a6cee3","#fb9a99","#984ea3","#ffff33")), ...)
}
##custom ggplot theme
theme_Publication <- function(...){
  theme_bw()+
    theme(plot.title = element_text(face = "bold",
                                    size = rel(1.2), hjust = 0.5),
          text = element_text(),
          panel.background = element_rect(colour = NA),
          plot.background = element_rect(colour = NA),
          panel.border = element_rect(colour = NA),
          axis.title = element_text(face = "bold",size = rel(1.3)),
          axis.title.y = element_text(angle=90,vjust =2),
          axis.title.x = element_text(vjust = -0.2),
          axis.text = element_text(size = rel(1.1)), 
          axis.line = element_line(colour="black"),
          axis.ticks = element_line(),
          panel.grid.major = element_line(colour="#f0f0f0"),
          panel.grid.minor = element_blank(),
          legend.key = element_rect(colour = NA),
          legend.position = "bottom",
          legend.direction = "horizontal",
          legend.key.size= unit(0.5, "cm"),
          legend.spacing = unit(0.05, "cm"),
          legend.text = element_text(size = rel(1)),
          legend.title = element_blank(),
          plot.margin=unit(c(10,5,5,5),"mm"),
          strip.background=element_rect(colour="#f0f0f0",fill="#f0f0f0"),
          strip.text = element_text(face="bold"))
}
```

### Study Area Map

``` r
#Load caribou herd data and clean up
herds.bc <- st_read(here::here("data", "caribouherds", "BC_Caribou_Range_CL.shp"))%>%
  mutate(HERD_NAME=as.character(HERD_NAME))%>%
  mutate(HERD_NAME=case_when(HERD_NAME%in%c("Scott", "Moberly")~"Klinse-Za", TRUE~HERD_NAME))%>%
  mutate(HERD_NAME=case_when(HERD_NAME%in%c("Klinse-Za") & STMTDCRBPP%in% "20-44"~"Scott", TRUE~HERD_NAME))%>%
  drop_na(RISK_STAT)%>%
  select(HERD_NAME)%>%
  st_transform(3005)

herds.ab <- st_read(here::here("data", "caribouherds", "Caribou_Range.shp"))%>%
  dplyr::rename(HERD_NAME=SUBUNIT)%>%
  mutate(HERD_NAME=as.character(HERD_NAME))%>%
  select(HERD_NAME)%>%
  st_transform(3005)%>%
  st_buffer(300)

herds <- rbind(herds.ab,herds.bc)%>%
  group_by(HERD_NAME)%>%
  summarize()

herds.cmg <- herds %>%
  filter(HERD_NAME%in%c("Burnt Pine", "Kennedy Siding", "Klinse-Za","Quintette","Narraway", "Jasper", "A La Peche","Little Smoky","Redrock-Prairie Creek"))%>%
  select(HERD_NAME)%>%
  st_transform(3005)

clip.extent <- extent(herds.cmg%>%filter(HERD_NAME%in%c("Burnt Pine", "Kennedy Siding", "Klinse-Za","Quintette","Narraway")))

herds <- herds%>%filter(!HERD_NAME%in%herds.cmg$HERD_NAME)


#Treaty 8 boundary
t8 <- st_read(here::here("data", "FN", "Traite_Pre_1975_Treaty_SHP.shp"))%>%
  filter(ENAME%in%c("Treaty #8 (1899)"))%>%
  st_transform(3005)

t8d <- st_read(here::here("data", "FN", "FNT_TRTY_A_polygon.shp"))%>%
  filter(TREATY%in%c("Treaty 8 - Disputed Area"))%>%
  st_transform(3005)

#First Nations Reserves
rv <- st_read(here::here("data", "FN", "AL_TA_BC_2_116_eng.shp"))%>%
  filter(NAME1%in%c("WEST MOBERLY LAKE 168A", "EAST MOBERLY LAKE 169"))%>%
  st_transform(3005)%>%
  select(NAME1)%>%
  mutate(lab=c("West Moberly First Nations", "Saulteau First Nations"))

#Administrative Boundaries
bord <- st_read(here::here("data", "borders", "North_America.shp"))%>%
  st_transform(3005)

lake <- st_read(here::here("data", "lakes", "ne_10m_lakes.shp"))%>%
  st_transform(3005)

#Elevation
dem.raster <- raster(here::here("data", "dem", "dem_KZ.tif"))


#Remove ocean
dem.raster[values(dem.raster)==0] <- NA

#Derive some topographic products
slope.raster <- terrain(dem.raster, opt='slope')
aspect.raster <- terrain(dem.raster, opt='aspect')
hill.raster <- hillShade(slope.raster, aspect.raster, 40, 270)

##make dataframe from rasters for ggplot plotting
dem<- dem.raster%>%
  aggregate(15)%>%
  as.data.frame(xy = TRUE)

hill<- hill.raster%>%
  aggregate(15)%>%
  as.data.frame(xy = TRUE)%>%
  drop_na(layer)

##make dataframe for smaller map
dem.small<- dem.raster%>%
  crop(extent(c(10.5E5,14E5,9.5E5,12.5E5)))%>%
  as.data.frame(xy = TRUE)

hill.small<- hill.raster%>%
  crop(extent(c(10.5E5,14E5,9.5E5,12.5E5)))%>%
  as.data.frame(xy = TRUE)

cmg.extent <- extent(c(10.5E5,14E5,9.5E5,12.5E5))%>%
  as('SpatialPolygons')%>%
  st_as_sf()%>%
  st_set_crs(st_crs(herds.cmg))

#PLOT
#Inset Map of western North America
westernNA <- ggplot()+
  geom_sf(data = bord, inherit.aes = FALSE, fill="grey70", color="black", size=0.1)+
  geom_sf(data = lake, inherit.aes = FALSE, fill="lightskyblue3", color="black", size=0.1)+
  geom_sf(data = herds, inherit.aes = FALSE, fill="black", color=NA, size=0.1, alpha=0.6)+
  geom_sf(data = herds.cmg, inherit.aes = FALSE, fill="black", color="grey30", size=0.1, alpha=0.2)+
  geom_sf(data = herds%>%rbind(herds.cmg)%>%filter(HERD_NAME%in%c("Columbia South", "Maligne", "Burnt Pine", "Purcells South", "South Selkirks", "Banff", "Allan Creek", "Duncan", "Purcell Central", "George Mtn", "Central Rockies", "Monashee", "Scott"))%>%st_buffer(10000), inherit.aes = FALSE, fill="brown", color=NA, alpha=0.8)+
  geom_sf(data = t8d, inherit.aes = FALSE, fill=NA, color="white", size=0.2, linetype="dashed")+
  geom_sf(data = t8, inherit.aes = FALSE, fill=NA, color="white", size=0.3)+
  geom_sf(data = cmg.extent, inherit.aes = FALSE, fill=NA, color="red", size=0.5, linetype="dotted")+
  scale_x_continuous(limits = c(2E5,20E5), expand = c(0, 0)) +
  scale_y_continuous(limits = c(1E5,17.2E5), expand = c(0, 0)) +
  guides(fill=FALSE, color=FALSE, alpha=FALSE)+
  theme(plot.margin=grid::unit(c(0,0,0,0), "mm"),
        axis.title=element_blank(),
        axis.text=element_blank(),
        axis.ticks=element_blank(),
        axis.ticks.length = unit(0, "pt"),
        panel.border = element_rect(colour = "black", fill=NA, size=2),
        panel.background = element_rect(fill = "lightskyblue3"))+
  coord_sf(datum = NA)+
  annotate("text", x = 17E5, y = 3E5, label = "USA", color="black", size=3)+
  annotate("text", x = 18E5, y = 10E5, label = "AB, CAN", color="black", size=2)+
  annotate("text", x = 13E5, y = 7E5, label = "BC, CAN", color="black", size=2)

##legend
leg <-ggplot()+
  geom_sf(data = herds%>%rbind(herds.cmg)%>%filter(HERD_NAME%in%c("Columbia South", "Maligne", "Burnt Pine", "Purcells South", "South Selkirks", "Banff", "Allan Creek", "Duncan", "Purcell Central", "George Mtn", "Central Rockies", "Monashee","Scott")), inherit.aes = FALSE, aes(fill="Extirpated"), alpha=0.5)+
  geom_sf(data = herds, inherit.aes = FALSE, aes(color="Central Mountain Group Caribou"), size=0.5, alpha=0.5)+
  geom_sf(data = herds.cmg, inherit.aes = FALSE, aes(fill="Mountain & Boreal Caribou"), size=1, alpha=0.2)+
  scale_x_continuous(limits = c(11.3E5,13.92E5), expand = c(0, 0)) +
  scale_y_continuous(limits = c(10.1E5,12.5E5), expand = c(0, 0))+
  scale_fill_manual(values=c("Extirpated"="brown",
                             "Mountain & Boreal Caribou"= "black"),
                    name="")+
  scale_color_manual(values=c("Central Mountain Group Caribou"="black"),
                     name="")+
  theme(legend.background = element_rect(colour = "transparent", fill ="transparent"),
        legend.key = element_rect(colour = "transparent", fill ="transparent"),
        panel.background = element_rect(fill = "transparent"), # bg of the panel
        plot.background = element_rect(fill = "transparent", color = NA), # bg of the plot
        legend.text = element_text(size = rel(1)),
        legend.title = element_blank(),
        legend.spacing = unit(0, "cm"), 
        legend.spacing.x = NULL,                 # Horizontal spacing
        legend.spacing.y = unit(0, "cm"),
        legend.position="bottom", 
        legend.box = "horizontal")+
  guides(
    color = guide_legend(order = 1),
    fill = guide_legend(order = 0)
  )

leg<- ggpubr::get_legend(leg)%>%
  as_ggplot()

ggsave(plot=leg,here::here("outputs", "SA_legend.png"), height=0.6, width=6.5)

#Study area map
sa.map <- ggplot()+
  geom_raster(data = dem.small, aes(x = x, y = y, fill = layer),alpha=0.5) +
  geom_raster(data = hill.small, aes(x = x, y = y, fill = layer, alpha=1-layer),fill="gray20") +
  geom_sf(data = bord, inherit.aes = FALSE, fill=NA, color="black", size=0.1)+
  geom_sf(data = lake, inherit.aes = FALSE, fill="lightskyblue3", color="black", size=0.1)+
  geom_sf(data = herds%>%rbind(herds.cmg)%>%filter(HERD_NAME%in%c("Maligne","Burnt Pine", "Purcells South", "South Selkirks", "Banff", "Allan Creek", "Duncan", "Purcell Central", "George Mtn", "Central Rockies", "Monashee","Scott")), inherit.aes = FALSE, fill="brown", color="brown", alpha=0.5)+
  geom_sf(data = herds%>%filter(!HERD_NAME%in%c("Maligne","Burnt Pine", "Purcells South", "South Selkirks", "Banff", "Allan Creek", "Duncan", "Purcell Central", "George Mtn", "Central Rockies", "Monashee","Scott")), inherit.aes = FALSE, fill="black", color="black", size=0.5, alpha=0.5)+
  geom_sf(data = herds%>%filter(HERD_NAME%in%c("Maligne","Burnt Pine", "Purcells South", "South Selkirks", "Banff", "Allan Creek", "Duncan", "Purcell Central", "George Mtn", "Central Rockies", "Monashee","Scott")), inherit.aes = FALSE, fill="black", color="black", size=0.5, alpha=0.2)+
  geom_sf(data = herds.cmg, inherit.aes = FALSE, fill="black", color="black", size=1, alpha=0.2)+
  geom_sf(data = rv%>%st_centroid(), inherit.aes = FALSE, fill="black", size=1.5)+
  scale_fill_gradientn(colours=brewer.pal(3,"YlGn")%>%rev(), na.value="light blue")+
  guides(fill=FALSE, color=FALSE, alpha=FALSE)+
  theme(plot.margin=grid::unit(c(0,0,0,0), "mm"),
        axis.title=element_blank(),
        axis.text=element_blank(),
        axis.ticks=element_blank(),
        panel.background = element_rect(fill = "white", colour = "grey50"))+
  scale_x_continuous(limits = c(10.5E5,14E5), expand = c(0, 0)) +
  scale_y_continuous(limits = c(9.5E5,12.5E5), expand = c(0, 0)) +
  #coord_sf(xlim = c(11E5,14E5),ylim = c(9.9E5,12.5E5))+
  annotation_custom(ggplotGrob(westernNA), xmin =10.3E5, xmax = 12.35E5, ymin = 9.55E5, ymax = 10.95E5)+
  geom_text(data = as.data.frame(herds.cmg%>%cbind(herds.cmg%>%st_centroid()%>%st_coordinates())), aes(X, Y, label = HERD_NAME), colour = "grey90",size=2.5)+
  ggrepel::geom_label_repel(
    data = rv,
    aes(label = str_wrap(lab,13), geometry = geometry),
    stat = "sf_coordinates",
    size=3,
    min.segment.length=unit(0,"lines"),
    segment.size  = 0.4,
    nudge_x=c(20000,20000),
    nudge_y=c(25000,-15000),
    hjust = 0,
    force=10
  )+
  annotation_scale(location = "br", width_hint = 0.25, text_col="grey90", text_cex = 0.8)+
  annotation_north_arrow(height = unit(1, "cm"), width = unit(1, "cm"),location = "tl", which_north = "true", style=north_arrow_orienteering(text_col="grey", text_size = 5))

ggsave(plot=sa.map,here::here("outputs", "cmg_map.png"), height=4, width=5)

##Plot Together
sa.map
```

![](README_files/figure-gfm/sa%20map-1.png)<!-- -->

### Partnership Agreement

``` r
####Partnership Agreement Area
pa <- st_read(here::here("data", "PA_fromJean", "Caribou_Zones_20200228.shp"))%>%
  st_transform(3005)

##existing parks
park <- st_read(here::here("data", "parks", "TA_PEP_SVW_polygon.shp"))%>%
  distinct(PROT_NAME)%>%
  st_crop(clip.extent)%>%
  st_transform(3005)

##pull out IPA
ipa <- pa%>%
  dplyr::filter(Zone%in%c("B2","B2 (Park)","B3","B3 (Park)"))%>%
  st_make_valid()%>%
  group_by(Date)%>%
  summarise()


##zones with decent security=A2,B2,B3,B4
pa <- pa%>%
  mutate(class=case_when(Zone%in%c("A2","B2 (Park)","B3 (Park)","B4", "B2", "B3", "B5")~"high",
                         Zone%in%c("A1","B1")~"moderate")%>%as.character())%>%
  st_make_valid()%>%
  group_by(class)%>%
  summarise()

##legend
leg <-ggplot()+
  geom_sf(data = pa%>%filter(class=="high"), inherit.aes = FALSE, aes(fill="High"), color=NA, size=0.1, alpha=0.7)+
  geom_sf(data = pa%>%filter(class=="moderate"), inherit.aes = FALSE, aes(fill="Moderate"), color=NA, size=0.1, alpha=0.7)+
  geom_sf(data = park, inherit.aes = FALSE, aes(fill="Park (pre-exisiting)"), color=NA, alpha=0.4)+
  scale_x_continuous(limits = c(11.3E5,13.92E5), expand = c(0, 0)) +
  scale_y_continuous(limits = c(10.1E5,12.5E5), expand = c(0, 0))+
  scale_fill_manual(values=c("High"="red",
                             "Moderate"="orange",
                             "Park (pre-exisiting)"="grey"),
                    name="Protection")+
  theme(legend.background = element_rect(colour = "transparent", fill = "transparent"),
        legend.key = element_rect(colour = "transparent", fill = "transparent"),
        legend.text = element_text(size = rel(1)),
        legend.title = element_text(face = "bold",size = rel(1.2)))

leg <- ggpubr::get_legend(leg)%>%
    as_ggplot()

##PLOT

PA <- ggplot()+
  geom_raster(data = dem.small, aes(x = x, y = y,fill = layer),alpha=0.5) +
  geom_raster(data = hill.small, aes(x = x, y = y, alpha=1-layer), fill="gray20") +
  geom_sf(data = bord, inherit.aes = FALSE, fill=NA, color="black", size=0.1)+
  geom_sf(data = lake, inherit.aes = FALSE, fill="lightskyblue3", color=NA, size=0.1)+
  geom_sf(data = pa%>%filter(class=="high"), inherit.aes = FALSE, fill="red", color=NA, size=0.1, alpha=0.7)+
  geom_sf(data = pa%>%filter(class=="moderate"), inherit.aes = FALSE, fill="orange", color=NA, size=0.1, alpha=0.7)+
  geom_sf(data = ipa, inherit.aes = FALSE, fill=NA, color="black", linetype="dashed", size=0.3)+
  geom_sf(data = herds, inherit.aes = FALSE, fill="black", color="black", size=0.5, alpha=0.5)+
  geom_sf(data = herds.cmg, inherit.aes = FALSE, fill=NA, color="black", size=0.6)+
  geom_sf(data = herds%>%
            rbind(herds.cmg)%>%
            filter(HERD_NAME%in%c("Maligne","Burnt Pine", "Purcells South", "South Selkirks", "Banff", "Allan Creek", "Duncan", "Purcell Central", "George Mtn", "Central Rockies", "Monashee","Scott")),
          inherit.aes = FALSE, fill=NA, color="brown", size=1)+
  geom_sf(data = park, inherit.aes = FALSE, fill="grey", color=NA, alpha=0.7)+
  geom_sf(data = rv%>%st_centroid(), inherit.aes = FALSE, fill="black", size=1.5)+
  scale_fill_gradientn(colours=brewer.pal(3,"YlGn")%>%rev(), na.value="light blue")+
  guides(fill=FALSE, color=FALSE, alpha=FALSE)+
  theme(plot.margin=grid::unit(c(0,0,0,0), "mm"),
        axis.title=element_blank(),
        axis.text=element_blank(),
        axis.ticks=element_blank(),
        panel.background = element_rect(fill = "white", colour = "grey50"))+
  scale_x_continuous(limits = c(11.3E5,13.92E5), expand = c(0, 0)) +
  scale_y_continuous(limits = c(10.1E5,12.5E5), expand = c(0, 0)) +
  annotation_custom(ggplotGrob(leg), xmin =11.4E5, xmax = 12.35E5, ymin = 10.05E5, ymax = 10.95E5)+
  geom_text(data = as.data.frame(herds.cmg%>%cbind(herds.cmg%>%st_centroid()%>%st_coordinates())), aes(X, Y, label = HERD_NAME), colour = "grey90",size=3)+
   ggrepel::geom_label_repel(
    data = rv,
    aes(label = str_wrap(lab,13), geometry = geometry),
    stat = "sf_coordinates",
    size=3,
    min.segment.length=unit(0,"lines"),
    segment.size  = 0.4,
    nudge_x=c(20000,20000),
    nudge_y=c(18000,-10000),
    hjust = 0,
    force=600
  )+
  annotation_scale(location = "br", width_hint = 0.25, text_col="grey90", text_cex = 0.8)+
  annotation_north_arrow(height = unit(1, "cm"), width = unit(1, "cm"),location = "tr", which_north = "true", style=north_arrow_orienteering(text_col="grey", text_size = 5))


####AREA PLOTS
##remove PA areas that were already park
park.clip <-park%>%
  st_union()%>%
  st_sf(Zone="P")%>%
  select(Zone)%>%
  rename("geometry"=".")

pa.herds <- st_read(here::here("data", "PA_fromJean", "Caribou_Zones_20200228.shp"))%>%
  mutate(Zone=as.character(Zone))%>%
  filter(!Zone %in% c("B2 (Park)","B3 (Park)"))%>%
  st_transform(3005)%>%
  st_make_valid()%>%
  st_difference(park.clip%>%st_buffer(5))%>%##remove parks
  select(Zone)

cmg.bc <- herds.cmg%>%
  st_intersection(bord%>%filter(FID_canada==9))%>%
  filter(HERD_NAME %in% c("Burnt Pine", "Kennedy Siding", "Klinse-Za","Quintette","Narraway"))

pa.herds <- pa.herds%>%
  rbind(park.clip)%>%
  st_intersection(cmg.bc)%>%##clip to bc
  mutate(area=st_area(.))%>%
  select(HERD_NAME,area,Zone)

unprotected <-st_difference(cmg.bc,pa.herds%>%st_union())%>%
  mutate(area=st_area(.),
         Zone="U")%>%
  select(HERD_NAME,area,Zone)


pa.herds <- pa.herds%>%
  rbind(unprotected)

#mapview(pa.herds["Zone"])
##make sure herd areas add up to total based of PA AREA
herd.areas <- cmg.bc%>%
  mutate(herd_area=st_area(.))%>%
  as_tibble()%>%
  select(-geometry)

# pa.herds%>%
#   as_tibble()%>%
#   ungroup()%>%
#   group_by(HERD_NAME)%>%
#   mutate(area=sum(as.numeric(area)))%>%
#   distinct(HERD_NAME, .keep_all=TRUE)

##add totals to pa.herds
pa.herds<- pa.herds%>%
  left_join(herd.areas, by="HERD_NAME")

##summarise
pa.herds <- pa.herds%>%
  group_by(HERD_NAME)%>%
  mutate(area_p=(as.numeric(area)/sum(as.numeric(area)))*100)%>%
  mutate(class=case_when(Zone%in%c("A2","B2","B3","B4","B5", "P")~"high",
                         Zone%in%c("A1","B1")~"moderate",
                         Zone%in%c("U")~"low")%>%as.character())

##fix labels
pa.herds <- pa.herds%>%
  mutate(Zone.l=case_when(Zone%in%c("A1", "B1")~"Extraction Reviewed",
                         Zone%in%c("A2")~"Extraction Moratorium",
                         Zone%in%c("B3")~"Park Expansion",
                         Zone%in%c("B2")~"Pre-existing Park Expansion",
                         Zone%in%c("B4")~"Restoration Focus",
                         Zone%in%c("B5")~"Indigenous Woodland",
                         Zone%in%c("P")~"Pre-existing Park",
                         Zone%in%c("U")~"Unprotected Land"))

##fix up factor levels
pa.herds <- pa.herds%>%ungroup()%>%
  mutate(class=fct_relevel(class, "high", "moderate", "low"),
         HERD_NAME=fct_relevel(HERD_NAME,"Klinse-Za", "Kennedy Siding", "Burnt Pine", "Quintette", "Narraway"),
         Zone.l=fct_relevel(Zone.l, "Extraction Moratorium","Indigenous Woodland","Park Expansion","Pre-existing Park","Restoration Focus", "Extraction Reviewed","Unprotected Land"))

##summary stats
##Area of high protection
pa.herds%>%
  filter(class%in%c("high"))%>%
  summarise(area=sum(area)/1E6)/
  (st_area(cmg.bc)/1E6)%>%as.numeric()%>%sum()*100

##Area of high protection that wasn't already a park
pa.herds%>%
  filter(Zone!="P")%>%
  filter(class%in%c("high"))%>%
  summarise(area=sum(area)/1E6)


###PLOT
areas <- ggplot(data=pa.herds, aes(x=class,y=area_p, fill=str_wrap(Zone.l,25)))+
  geom_col()+
  facet_wrap(vars(HERD_NAME))+
  ylab("Percent area")+
  xlab("Protection")+
  labs(fill="Zone")+
  theme_Publication()+
  scale_fill_brewer(palette = "Set2")+
  theme(legend.position = c(0.85,0.255),
        legend.direction = "vertical",
        legend.key.size= unit(0.5, "cm"),
        legend.spacing = unit(0.05, "cm"),
        legend.text = element_text(size = rel(0.7)),
        legend.title = element_text(size = rel(0.9)),
         legend.background = element_rect(fill = "white"),
         legend.box.background = element_rect(fill = "white"),
        axis.text.x=element_text(angle=45, vjust=1, hjust=1))


##PLOT TOGETHER
pa.map <- ggarrange(PA, areas, ncol = 2, nrow = 1, labels="AUTO")

ggsave(plot=pa.map,here::here("outputs", "PA_map.png"), height=5, width=10)

pa.map
```

![](README_files/figure-gfm/PA%20map-1.png)<!-- -->

``` r
##table of PA
pa.table <-pa.herds%>%
            select(HERD_NAME, herd_area, Zone, Zone.l,area_p, area,class)%>%
            mutate(herd_area=round(herd_area/1E6,0)%>%as.numeric(),
                   area_p=round(area_p,1)%>%as.numeric(),
                   area=round(area/1E6,0)%>%as.numeric())%>%
            arrange(HERD_NAME, Zone,Zone.l)%>%
            as_tibble()%>%
            select(-geometry)%>%
            rename(Population=HERD_NAME,
                   "Area (%)"=area_p,
                   "Population area (km2)"=herd_area,
                   "Area (km2)"=area,
                   Class=class,
                   Description=Zone.l)

write_csv(pa.table,here::here("outputs", "PA_areas_byherd.csv"))
```

### Klinse-Za Population Trend- From Integrated Population Model of McNay et al 2020

``` r
###load data
df<-read_csv(here::here("data","abundance_MF.csv"))


##plot PRE
ggplot(data=df%>%filter(herd%in%"Klinse-Za" & yrs<2014),aes(x=yrs,y=est))+
  geom_ribbon(alpha=0.2, aes(ymin=lower, ymax=upper))+
  geom_line() +
  geom_point(color="grey50", size=1) +
  ggtitle("Klinse-Za Population Trend")+
  scale_y_continuous(limits = c(0,400), expand = c(0, 0))+
  theme_bw()+
  xlab("Year")+
  ylab("Population estimate")+
  geom_label_repel(data=data.frame(label=c("Indigenous-led\nrecovery starts"),
                                   yrs=c(2013),
                                   est=c(38))%>%
                     mutate(label=as.factor(label)),
                   aes(x=yrs,y=est, label=label),
                   size=3,
                   min.segment.length=unit(0,"lines"),
                   segment.size  = 0.3,
                   segment.color = "grey80",
                   nudge_y = 100,
                   nudge_x=-3,
                   hjust = 0,
                   force=0.1
  )
```

![](README_files/figure-gfm/Pop%20trend%20plot-1.png)<!-- -->

``` r
ggsave(here::here("outputs", "kz_trend_pre.png"), height=4, width=4)



##plot ALL
ggplot(data=df%>%filter(herd%in%"Klinse-Za"),aes(x=yrs,y=est))+
  geom_ribbon(alpha=0.2, aes(ymin=lower, ymax=upper))+
  geom_line() +
  geom_point(color="grey50", size=1) +
  ggtitle("Klinse-Za Population Trend",subtitle = "Recovery following Indigenous-led actions")+
  scale_y_continuous(limits = c(0,400), expand = c(0, 0))+
  theme_bw()+
  xlab("Year")+
  ylab("Population estimate")+
  geom_label_repel(data=data.frame(label=c("Partnership Agreement\nsigned","Indigenous-led\nrecovery starts"),
                                 yrs=c(2020,2013),
                                 est=c(88,38))%>%
                     mutate(label=as.factor(label)),
                   aes(x=yrs,y=est, label=label),
    size=3,
    min.segment.length=unit(0,"lines"),
    segment.size  = 0.3,
    segment.color = "grey80",
    nudge_y = 100,
    nudge_x=-3,
    hjust = 0,
    force=0.1
  )
```

![](README_files/figure-gfm/Pop%20trend%20plot-2.png)<!-- -->

``` r
ggsave(here::here("outputs", "kz_trend.png"), height=4, width=4)
```

### Plot Disturbances

``` r
#Load disturbances
mines <- st_read(here::here("data", "HumanFootprints", "CL_mines.shp"))%>%
  st_transform(herds.cmg%>%st_crs())

cb <- st_read(here::here("data", "cutblocks","VEG_CONSOLIDATED_CUT_BLOCKS_SP","CNS_CUT_BL_polygon.shp"))%>%
  st_transform(herds.cmg%>%st_crs())

cb.clip <- cb%>%
  st_make_valid()%>%
  st_crop(extent(cmg.bc))%>%
  st_intersection(cmg.bc%>%st_make_valid())

b.yr <- cb.clip%>%
  mutate(area=st_area(.))%>%
  group_by(HARVESTYR,HERD_NAME)%>%
  summarise(cut=(sum(area)%>%as.numeric())/1E6)%>%
  as_tibble()%>%
  left_join(herd.areas)%>%
  mutate(herd_area=(herd_area%>%as.numeric())/1E6)%>%
  mutate(cut_p=(cut/herd_area)*100)

##plot annual area logged through time
ggplot(data=b.yr%>%filter(HARVESTYR>1980), aes(x=HARVESTYR,y=cut_p))+
  geom_col()+
  facet_wrap(vars(HERD_NAME))+
  ylab("Percent area logged")+
  xlab("Year")+
  labs(fill="Zone")+
  theme_Publication()+
  scale_fill_brewer(palette = "Set2")+
  theme(legend.position = c(0.85,0.255),
        legend.direction = "vertical",
        legend.key.size= unit(0.5, "cm"),
        legend.spacing = unit(0.05, "cm"),
        legend.text = element_text(size = rel(0.7)),
        legend.title = element_text(size = rel(0.9)),
        legend.background = element_rect(fill = "white"),
        legend.box.background = element_rect(fill = "white"),
        axis.text.x=element_text(angle=45, vjust=1, hjust=1))
```

![](README_files/figure-gfm/Plot%20Disturbance-1.png)<!-- -->

``` r
##PLOT entire CMG
c.rast <- cb%>%
  filter(HARVESTYR%in%1980:2012)%>%
  mutate(count=1)%>%
  fasterize(raster=dem.raster%>%
              crop(extent(cmg.extent)),field="count", fun="sum")%>%
  as.data.frame(xy = TRUE)%>%
  drop_na(layer)

c.rast.new <- cb%>%
  filter(HARVESTYR%in%2013:2020)%>%
  mutate(count=1)%>%
  fasterize(raster=dem.raster%>%
              crop(extent(cmg.extent)),field="count", fun="sum")%>%
  as.data.frame(xy = TRUE)%>%
  drop_na(layer)



rd.rast <- raster(here::here("data", "road", "rd_use.tif"))%>%
  aggregate(3)
  
rd <- raster::as.data.frame(rd.rast, xy = TRUE)
rd <- rd%>%drop_na(rd_use)%>%filter(rd_use>0)


##legend
leg.dist <-ggplot()+
  geom_line(data = rd, aes(x = x, y = y, color="Road"))+
  geom_tile(data = c.rast, aes(x = x, y = y, fill="Logging (1980-2012)"))+
  geom_tile(data = c.rast.new, aes(x = x, y = y, fill="Logging (2013-2019)"))+
  geom_sf(data = mines, inherit.aes = FALSE, aes(fill="Mines"))+
  geom_sf(data = park.clip, inherit.aes = FALSE, aes(fill="Park"), color=NA, alpha=0.6)+
  scale_x_continuous(limits = c(11.3E5,13.92E5), expand = c(0, 0)) +
  scale_y_continuous(limits = c(10.1E5,12.5E5), expand = c(0, 0))+
  scale_fill_manual(values=c("Park"="grey",
                             "Mines"= "#f7fcb9",
                             "Logging (1980-2012)"="orange",
                             "Logging (2013-2019)"="red"),
                    name="")+
  scale_color_manual(values=c("Road"="grey35"),
                     name="")+
  theme(legend.background = element_rect(colour = "transparent", fill ="transparent"),
        legend.key = element_rect(colour = "transparent", fill ="transparent"),
        panel.background = element_rect(fill = "transparent"), # bg of the panel
        plot.background = element_rect(fill = "transparent", color = NA), # bg of the plot
        legend.text = element_text(size = rel(1)),
        legend.title = element_blank(),
        legend.spacing = unit(0, "cm"), 
        legend.spacing.x = NULL,                 # Horizontal spacing
        legend.spacing.y = unit(0, "cm"),
        legend.position="bottom", 
        legend.box = "horizontal")+
  guides(
    color = guide_legend(order = 0),
    fill = guide_legend(order = 1)
  )

leg.dist <- ggpubr::get_legend(leg.dist)%>%
  as_ggplot()

ggsave(plot=leg.dist,here::here("outputs", "disturbance_legend.png"), height=0.6, width=6.2)

ch <- st_read(here::here("data", "CH_638_Rangifer_tarandus_caribou_SouthMountain", "CH_638_Rangifer_tarandus_caribou_SouthMountain.shp"))%>%
  st_make_grid()%>%
  st_transform(3005)%>%
  st_crop(extent(herds.cmg))

ggplot()+
  geom_raster(data = dem.small, aes(x = x, y = y,fill = layer),alpha=0.5) +
  geom_raster(data = hill.small, aes(x = x, y = y, alpha=1-layer), fill="gray20") +
  geom_sf(data = bord, inherit.aes = FALSE, fill=NA, color="black", size=0.1)+
  geom_sf(data = lake, inherit.aes = FALSE, fill="lightskyblue3", color=NA, size=0.1)+
  geom_tile(data = rd, aes(x = x, y = y), fill="grey20", width = 300, height=300)+
  geom_tile(data = c.rast, aes(x = x, y = y),fill="orange")+
  geom_tile(data = c.rast.new, aes(x = x, y = y),fill="red")+
  geom_sf(data = mines, inherit.aes = FALSE, fill="#f7fcb9", color=NA)+
  #geom_sf(data = ch, inherit.aes = FALSE, fill="grey", color="grey40", alpha=0.3,size=0.1)+
  geom_sf(data = park.clip, inherit.aes = FALSE, fill="grey", color=NA, alpha=0.6)+
  geom_sf(data = herds, inherit.aes = FALSE, fill="black", color="black", size=0.5, alpha=0.5)+
  geom_sf(data = herds.cmg, inherit.aes = FALSE, fill=NA, color="black", size=1)+
  geom_sf(data = herds%>%
            rbind(herds.cmg)%>%
            filter(HERD_NAME%in%c("Maligne","Burnt Pine", "Purcells South", "South Selkirks", "Banff", "Allan Creek", "Duncan", "Purcell Central", "George Mtn", "Central Rockies", "Monashee","Scott")),
          inherit.aes = FALSE, fill=NA, color="brown", size=1)+
  geom_sf(data = rv%>%st_centroid(), inherit.aes = FALSE, fill="black", size=1.5)+
  #scale_fill_gradientn(colours=brewer.pal(8,"YlGn")%>%rev(), na.value="light blue")+
  scale_fill_gradientn(colours=paletteer_dynamic("cartography::green.pal",n=10)%>%rev(), na.value="light blue")+
  guides(fill=FALSE, color=FALSE, alpha=FALSE)+
  theme(plot.margin=grid::unit(c(0,0,0,0), "mm"),
        axis.title=element_blank(),
        axis.text=element_blank(),
        axis.ticks=element_blank(),
        panel.background = element_rect(fill = "white", colour = "grey50"))+
  scale_x_continuous(limits = c(11.3E5,13.92E5), expand = c(0, 0)) +
  scale_y_continuous(limits = c(10.1E5,12.45E5), expand = c(0, 0))+
  annotation_scale(location = "br", width_hint = 0.25, text_col="grey", text_cex = 0.8)+
  annotation_north_arrow(height = unit(1, "cm"), width = unit(1, "cm"),location = "tr", which_north = "true", style=north_arrow_orienteering(text_col="grey", text_size = 5))
```

![](README_files/figure-gfm/Plot%20Disturbance-2.png)<!-- -->

``` r
ggsave(here::here("outputs", "disturbance_map1.png"), height=4.5, width=5)


##PLOT KZ ONLY
c.rast.small <- cb%>%
  filter(HARVESTYR%in%1980:2012)%>%
  mutate(count=1)%>%
  fasterize(raster=rd.rast,field="count", fun="sum")%>%
  as.data.frame(xy = TRUE)%>%
  drop_na(layer)

c.rast.new.small <- cb%>%
  filter(HARVESTYR%in%2013:2020)%>%
  mutate(count=1)%>%
  fasterize(raster=rd.rast,field="count", fun="sum")%>%
  as.data.frame(xy = TRUE)%>%
  drop_na(layer)



ggplot()+
  geom_raster(data = dem.small, aes(x = x, y = y,fill = layer),alpha=0.5) +
  geom_raster(data = hill.small, aes(x = x, y = y, alpha=1-layer), fill="gray20") +
  geom_sf(data = bord, inherit.aes = FALSE, fill=NA, color="black", size=0.1)+
  geom_sf(data = lake, inherit.aes = FALSE, fill="lightskyblue3", color=NA, size=0.1)+
  geom_tile(data = rd, aes(x = x, y = y), fill="grey20", width = 300,height=300)+
  geom_tile(data = c.rast.small, aes(x = x, y = y),fill="orange")+
  geom_tile(data = c.rast.new.small, aes(x = x, y = y),fill="red")+
  geom_sf(data = mines, inherit.aes = FALSE, fill="#f7fcb9", color=NA)+
  #geom_sf(data = ch, inherit.aes = FALSE, fill="grey", color="grey40", alpha=0.3,size=0.1)+
  geom_sf(data = park.clip, inherit.aes = FALSE, fill="grey", color=NA, alpha=0.6)+
  geom_sf(data = herds, inherit.aes = FALSE, fill="black", color="black", size=0.5, alpha=0.5)+
  geom_sf(data = herds.cmg, inherit.aes = FALSE, fill=NA, color="black", size=1)+
  geom_sf(data = herds%>%
            rbind(herds.cmg)%>%
            filter(HERD_NAME%in%c("Maligne","Burnt Pine", "Purcells South", "South Selkirks", "Banff", "Allan Creek", "Duncan", "Purcell Central", "George Mtn", "Central Rockies", "Monashee","Scott")),
          inherit.aes = FALSE, fill=NA, color="brown", size=1)+
  geom_sf(data = rv%>%st_centroid(), inherit.aes = FALSE, fill="black", size=1.5)+
  #scale_fill_gradientn(colours=brewer.pal(8,"YlGn")%>%rev(), na.value="light blue")+
  scale_fill_gradientn(colours=paletteer_dynamic("cartography::green.pal",n=10)%>%rev(), na.value="light blue")+
  guides(fill=FALSE, color=FALSE, alpha=FALSE)+
  theme(plot.margin=grid::unit(c(0,0,0,0), "mm"),
        axis.title=element_blank(),
        axis.text=element_blank(),
        axis.ticks=element_blank(),
        panel.background = element_rect(fill = "white", colour = "grey50"))+
  scale_x_continuous(limits = c(11.35E5,12.53E5), expand = c(0, 0)) +
  scale_y_continuous(limits = c(11.35E5,12.43E5), expand = c(0, 0))
```

![](README_files/figure-gfm/Plot%20Disturbance-3.png)<!-- -->

``` r
ggsave(here::here("outputs", "disturbance_map2.png"), height=4.6, width=5)
```

### Summary stats

``` r
###how many hectatres cut during the KZ caribou recovery actions ongoing?
st_erase = function(x, y) st_difference(x%>%st_make_valid(), st_union(st_combine(y))%>%st_make_valid())

cb%>%
  filter(HARVESTYR%in%2013:2020)%>%
  st_intersection(herds.cmg%>%filter(HERD_NAME%in%"Klinse-Za"))%>%
  mutate(area=units::set_units(st_area(.),"ha"))%>%
  summarise(sum=sum(area))
```

    ## Simple feature collection with 1 feature and 1 field
    ## geometry type:  MULTIPOLYGON
    ## dimension:      XY
    ## bbox:           xmin: 1137065 ymin: 1143034 xmax: 1239390 ymax: 1242020
    ## epsg (SRID):    3005
    ## proj4string:    +proj=aea +lat_1=50 +lat_2=58.5 +lat_0=45 +lon_0=-126 +x_0=1000000 +y_0=0 +ellps=GRS80 +towgs84=0,0,0,0,0,0,0 +units=m +no_defs
    ##             sum                       geometry
    ## 1 11026.36 [ha] MULTIPOLYGON (((1150057 119...

``` r
cb%>%
  filter(HARVESTYR%in%2013:2020)%>%
  st_intersection(herds.cmg%>%filter(HERD_NAME%in%"Klinse-Za"))%>%
  mutate(area=units::set_units(st_area(.),"ha"))%>%
  tibble()%>%
  group_by(HARVESTYR)%>%
    summarise(sum=sum(area)%>%as.numeric())%>%
  ggplot(aes(x=HARVESTYR,y=sum))+
  geom_col()+
  ylab("Percent area logged")+
  xlab("Year")+
  theme_Publication()+
  scale_fill_brewer(palette = "Set2")+
  theme(legend.position = c(0.85,0.255),
        legend.direction = "vertical",
        legend.key.size= unit(0.5, "cm"),
        legend.spacing = unit(0.05, "cm"),
        legend.text = element_text(size = rel(0.7)),
        legend.title = element_text(size = rel(0.9)),
        legend.background = element_rect(fill = "white"),
        legend.box.background = element_rect(fill = "white"),
        axis.text.x=element_text(angle=45, vjust=1, hjust=1))
```

![](README_files/figure-gfm/summary%20stats-1.png)<!-- -->

``` r
##check another way, yes all good 11021.46 [ha]
# a <-herds.cmg%>%filter(HERD_NAME%in%"Klinse-Za")%>%
#   st_erase(cb%>%
#   filter(HARVESTYR%in%2013:2020))
# units::set_units(st_area(herds.cmg%>%filter(HERD_NAME%in%"Klinse-Za")),"ha")-units::set_units(st_area(a),"ha")
  

## Partnership Agreement table
kable(pa.table)
```

| Population     | Population area (km2) | Zone | Description                 | Area (%) | Area (km2) | Class    |
| :------------- | --------------------: | :--- | :-------------------------- | -------: | ---------: | :------- |
| Klinse-Za      |                  5506 | A1   | Extraction Reviewed         |      0.2 |         10 | moderate |
| Klinse-Za      |                  5506 | A2   | Extraction Moratorium       |     21.2 |       1167 | high     |
| Klinse-Za      |                  5506 | B1   | Extraction Reviewed         |     15.9 |        876 | moderate |
| Klinse-Za      |                  5506 | B2   | Pre-existing Park Expansion |      5.8 |        319 | high     |
| Klinse-Za      |                  5506 | B3   | Park Expansion              |     30.4 |       1672 | high     |
| Klinse-Za      |                  5506 | B4   | Restoration Focus           |      5.8 |        321 | high     |
| Klinse-Za      |                  5506 | B5   | Indigenous Woodland         |      4.4 |        243 | high     |
| Klinse-Za      |                  5506 | P    | Pre-existing Park           |      1.8 |        100 | high     |
| Klinse-Za      |                  5506 | U    | Unprotected Land            |     14.5 |        799 | low      |
| Kennedy Siding |                  2962 | A1   | Extraction Reviewed         |      0.0 |          1 | moderate |
| Kennedy Siding |                  2962 | A2   | Extraction Moratorium       |     34.2 |       1012 | high     |
| Kennedy Siding |                  2962 | B1   | Extraction Reviewed         |      3.3 |         98 | moderate |
| Kennedy Siding |                  2962 | B3   | Park Expansion              |      0.3 |         10 | high     |
| Kennedy Siding |                  2962 | P    | Pre-existing Park           |     14.6 |        433 | high     |
| Kennedy Siding |                  2962 | U    | Unprotected Land            |     47.5 |       1408 | low      |
| Burnt Pine     |                   710 | A1   | Extraction Reviewed         |      0.4 |          3 | moderate |
| Burnt Pine     |                   710 | A2   | Extraction Moratorium       |     21.4 |        152 | high     |
| Burnt Pine     |                   710 | B1   | Extraction Reviewed         |     14.2 |        101 | moderate |
| Burnt Pine     |                   710 | U    | Unprotected Land            |     64.0 |        454 | low      |
| Quintette      |                  6078 | A1   | Extraction Reviewed         |      2.2 |        132 | moderate |
| Quintette      |                  6078 | A2   | Extraction Moratorium       |     18.1 |       1098 | high     |
| Quintette      |                  6078 | P    | Pre-existing Park           |      9.9 |        602 | high     |
| Quintette      |                  6078 | U    | Unprotected Land            |     69.9 |       4247 | low      |
| Narraway       |                  6372 | A1   | Extraction Reviewed         |      0.6 |         41 | moderate |
| Narraway       |                  6372 | A2   | Extraction Moratorium       |     16.5 |       1049 | high     |
| Narraway       |                  6372 | P    | Pre-existing Park           |     20.2 |       1289 | high     |
| Narraway       |                  6372 | U    | Unprotected Land            |     62.7 |       3992 | low      |

``` r
##size of new conservation area overlapping with CMG herds
pa.table%>%
  filter(!Zone%in%c("B2","U","P"))%>%
  summarise(area=sum(`Area (km2)`))%>%
  as.numeric()%>%
  print()
```

    ## [1] 7986
