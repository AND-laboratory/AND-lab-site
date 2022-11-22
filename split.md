---
layout: page
title: Splitting datasets
subtitle: Example code for splitting datasets for reproducibility
---

### The ABCD Study

The [Adolescent Brain Cognitive Development (ABCD) Study](https://abcdstudy.org/) is a longituindal ongoing cohort of over 11,000 children in the US across multiple research sites.
The ongoing and intermittent release of data means that the whole sample may not have all measures available at any one time, however, this presents an interesting opportunity to arrange training and holdout samples.

For example, we may want to build a latent model of measures that correlate first using an exploratory factor analysis with one subsample, repeat that latent structure in a second (or more) confirmatory factor analysis, and then use those latent scores to predict some other outome in a completely separate experimental sample.

Once ABCD data is downloaded from the NDA website, you will have a number of datasets depending on your variables of interest, that you will need to merge. As an example, I name two of these "dataset1" and "dataset2" in the code below. "dataset1" should be the data you need to build the model. 

All ABCD datasets use "src_subject_id" as the ID variable and "eventname" as the wave. This example is in R, and is for an intermittent release where not all of the 2 year follow up had been released from the full sample (only partial release). In our analyses, we wanted to build a latent structure of adversity data from baseline, and then use that latent structure to predict trajectories in anxiety symptoms over three waves of data. Therefore, the model building sample only needed baseline data (everyone in the study was available), and the experimental sample needed to be made up of people that had three waves available at the time of release (i.e., people that had a row of "eventname" = "2_year_follow_up_y_arm_1"). The data is in long format (each observation on a row, multiple rows per ID).


```{r abcd_split}
# ----------------- Model Building Sample -----------------

all_data <- merge(dataset1, dataset2, by = c("src_subject_id","eventname"), 
                  all = TRUE, sort = TRUE)

# 'model_building' variable is TRUE if it's model building (and FALSE if it's experimental sample AND in a 2 year follow up row)
# we also wanted to make sure our outcome of interest (anxiety) was not missing.

all_data$model_building <- all_data$eventname=='2_year_follow_up_y_arm_1' & 
  is.na(all_data$cbcl_scr_dsm5_anxdisord_t)

# These do NOT have 2 year follow up anxiety data, therefore they are for model building:

model_building_2y <- all_data[ which(all_data$model_building==TRUE), ]

# just check no duplicate IDs in there!

check_MB_IDs <- unique(model_building_2y$src_subject_id)

#create just IDs for MB because we don't want the 2 yr blank data in there:

model_building_IDs <- select(model_building_2y, "src_subject_id")

model_building <- merge(model_building_IDs,dataset1,by="src_subject_id",all.x=TRUE)
write.csv(model_building, file = "model_building_aces.csv")

# You now have a dataset, called "model_building" for building a latent model. 
# You can further split this into random subsamples if you want to do things like an EFA and then test a few CFAs (especially after needing to tweak things) to make sure the latent structure is robust:

# Note - at this stage we cleaned up some of the unecessary variables manually and loaded them back in. You can do this, or leave it how it is.

MB_final <- read_csv("C:/Users/michelle/Google Drive/Professional/Students/0_Monash Students/Chrysa Tsiapas/model_building_aces_final.csv")
MB_final_table <- as.data.table(MB_final)

# Split model building dataset into 3 random subsamples:

MB_final$subsample <- sample(factor(rep(1:3, length.out=nrow(MB_final)), 
                                 labels=paste0("subsample", 1:3)))
MB_subsample1 <- MB_final[which(MB_final$subsample=="subsample1"), ]
MB_subsample2 <- MB_final[which(MB_final$subsample=="subsample2"), ]
MB_subsample3 <- MB_final[which(MB_final$subsample=="subsample3"), ]

write.csv(MB_subsample1, file = "MB_subsample1.csv")
write.csv(MB_subsample2, file = "MB_subsample2.csv")
write.csv(MB_subsample3, file = "MB_subsample3.csv")

# ----------------- Experimental Sample -----------------

# Don't use the FALSE 'model_building' value because this will include baseline and 1 year follow up rows for model building IDs:

exp_sample_full <- all_data[ which(all_data$eventname==
                                     '2_year_follow_up_y_arm_1' &
                                     !is.na(anxiety_full$cbcl_scr_dsm5_anxdisord_t)), ]
exp_sample_IDs <- select(exp_sample_full, "src_subject_id")
check_ES_IDs <- unique(exp_sample_IDs$src_subject_id)

# just keep rows where the ID is in the experimental sample ID list made above:

exp_sample <- all_data[which(all_data$src_subject_id %in% exp_sample_full$src_subject_id), ]
length(unique(exp_sample$src_subject_id))
write.csv(exp_sample, file = "experimental_sample.csv")


```