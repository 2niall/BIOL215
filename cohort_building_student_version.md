
# Homework 3: Cohort Building

### BIOMEDIN 215 (Data Driven Medicine), Fall 2017 

### Due: Tuesday, October 24, 2017 

In this assignment you will gain experience extracting and transforming clinical data into datasets for downstream statistical analysis. You will practice using
common time-saving tools in the R programming language that are ideally suited to these tasks.

You will use the [MIMIC III database](https://mimic.physionet.org/mimictables/patients/) as a sandbox to create a dataset describing a cohort of patients admitted to the Intensive Care Unit of the Beth Israel Deaconess Medical Center in Boston, Massachusetts. You will analyze this cohort to identify patients that undergo septic shock during their admission. We will be following along with the cohort building process presented in ["A targeted real-time early warning score (TREWScore) for septic shock" by Henry et al.](http://stm.sciencemag.org/content/7/299/299ra122.full) published in Science Translational Medicine in 2015. We will also be referring to [a cited paper by Angus et al](https://github.com/MIT-LCP/mimic-code/blob/master/concepts/sepsis/angus2001.pdf). All of the data you need for this assignment is available on Canvas.

Please edit this document directly using either Jupyter or R markdown in RStudio and answer each of the questions in-line. Jupyter and R markdown are useful tools for reproducible research that you will use over and over again in your later work. They are worth taking the short amount of time necessary to learn them. Turn in a single .pdf document showing all of your code and output for the entire assignment, with each question clearly demarcated. Submit your completed assignment through Canvas.

## 0. Getting Ready
The first thing we need to do is load all of the packages we will use for this assignment. Please load the packages `dplyr`, `tidyr`, `lubridate`, `stringr`, `readr`, and `ggplot2`. Also, please run the command `Sys.setenv(TZ='UTC')`. Without it, your date-time data will misbehave.

## 1. Building a Cohort Based on Inclusion Criteria and Defining Endpoints
### Loading Data

#### 1.1 (5 pts)

The first part of any patient-level study is to identify a cohort of patients who are relevant to the study, and at what point during their records they became eligible. Typically, this is done with a set of "inclusion critera", which, if met, qualify the patient for inclusion in the cohort. 

In our study, we will consider the inclusion criteria used in the TREWScore paper.

Read the first paragraph of the *Materials and Methods - Study Design* in the TREWScore paper. What criteria did the authors use to determine which patients should enter the study?

#### 1.2 (5 pts)

Once you have found the inclusion criteria, take a look at the [MIMIC documentation](https://mimic.physionet.org/about/mimic/) and report which table(s) in MIMIC you would query in order to identify patients that meet the inclusion criteria. If you're stuck, try looking through the [mimic-code repository](https://github.com/MIT-LCP/mimic-code) provided by the MIMIC maintainers to get some ideas as to how to query the database.

#### 1.3 (2 pts) 

It can be tricky to develop the SQL queries necessary to extract the cohort of interest. Fortunately, the course staff ran the necessary query on the MIMIC III database to extract the identifiers of patients that meet the inclusion criteria discussed above. 

Read the vitals and labs data for our cohort stored in *vitals_cohort_sirs.csv* and *labs_cohort* into R dataframes. Since these CSV files are moderately sized, we suggest using a function from the [readr](https://cran.r-project.org/web/packages/readr/readr.pdf) package to load the data.

Once you have loaded the files into R dataframes, call `head` and `str` on each dataframe to get a feel for what is contained in each column

#### 1.4 (5 pts)

While we are ultimately interested in utilizing all of the data from all of the patients that meet our inclusion criteria, it is good practice to work with a small *development set* of data for the purposes of developing our analytical pipeline, so that we may test ideas quickly without having to wait for code to execute on large datasets.

Using `dplyr` commands, extract the *subject_id* of the 1000 subjects with the smallest *subject_id*s and filter the labs and vitals dataframes to contain only the subjects in that set of 1000.

**Note:** We will continue to work with this set of 1000 subjects throughout the remainder of the assignment. From this point forward, feel free to use an even smaller set to experiment with costly data transformations.

#### 1.5 (5 pts) 

The Systemic Inflammatory Response Syndrome (SIRS) criteria has been an integral tool for the clinical definition of sepsis for the past several decades. In the TREWScore paper, the authors considered a patient to have sepsis if at least two of the four SIRS criteria were simultaneously met during an admission where a suspicion of infection was also present.

The four SIRS criteria are as follows:
1. Temperature > 38&deg;C or < 36&deg;C
2. Heart Rate > 90
3. Respiratory Rate > 20 or PaCO$_{2}$< 32mmHg
4. WBC > 12,000/$mm^{3}$, < 4000/$mm^{3}$, or > 10% bands

You may read more about SIRS (and some recent associated controversies surrounding its use) at https://www.ncbi.nlm.nih.gov/pubmed/1303622 and http://www.nejm.org/doi/full/10.1056/NEJMoa1415236#t=article.

The next step in our process will be to assess whether patients satisfy each of the SIRS criteria at each time step that vitals or lab data is available. To this end, we would like to have a dataframe where each row corresponds to a unique combination of *subject_id*, *hadm_id*, *icustay_id*, and *charttime*, and with one column for each unique type of lab or vital that was measured at that time. This may seem fairly complicated at first, but there is a relatively simple approach that leverages `dplyr` and `tidyr`. Let's walk through it step-by-step to build some intuition.

First, use the `spread` operation from `tidyr` on each of the vitals and labs dataframes to create a column for each type of lab/vital. 

You should see an error. Give a possible reason as to why this error might have occurred and propose at least one solution (in prose, not code).

#### 1.6 (5 pts) 
When working with EHR data we occasionally need to make *ad hoc* choices that allow us to proceed with our analysis. Here we make one such choice in working around the error we saw above. 

To solve the above error, use the `group_by`, `summarise` and `ungroup` commands from `dplyr` to implement your solution.

After you have solved the error, try to spread on *lab_id* and *vital_id* again and  use `str` to inspect the resulting dataframes.

#### 1.7. (5 pts)

Since the measurement times for the vital signs may be different from those of the labs, the next step is to merge the vitals and labs dataframes together to get the full timeline for each patient. 

With `full_join`, merge the spread labs and vitals dataframes you generated previously, using the common columns in the two dataframes.

#### 1.8. (5 pts)

You will notice that the resulting dataframe contains a lot of "missing" values recorded as `NA`. There are many potential approaches for handling missing values that we could take to address this issue. In this case, we are going to use a last-value-carried-forward approach within an ICU stay to fill in missing values.

In a sentence or two, discuss any potential benefits and drawbacks of this approach. After that, implement this strategy by sorting on *subject_id, hadm_id, icustay_id*, and *charttime*, performing an appropriate `group_by` call and then using the `fill` function from `tidyr`.

#### 1.9 (5 pts)
Now we have a record of the most recent value for each lab or vital within an ICU stay for each patient in our development set. From this data, create a new dataframe called *SIRS* that has a record for each row in your timeline dataframe developed previously and a column indicating whether each of the SIRS criteria were satisfied at each chart time, and a final column indicating whether at least 2 of the SIRS criteria were satisfied. Assume that if a value is unknown that the patient does not meet that SIRS criterion.

#### 1.10 (5 pts)

Let’s visualize what we’ve done so far. This should help you identify any errors that you might have made. It’s often difficult to keep track of all the data manipulation, even for expert researchers! That’s why it’s good practice to plot things every once in a while as a sanity check.

For the patient with subject_id = 3, use `ggplot` and `facet_wrap` to plot the trajectories for each of the labs and vitals in their timeline. Plot the datetime on the x-axis and the recorded measurement on the y-axis. Color the plotted points by whether or not the patient meets two or more of the SIRS criteria at that timepoint.

#### 1.11 (3 points)

At this point, we have computed the SIRS criteria for every patient in our development set. Now it's time to determine which patients had suspicion of infection. 

Find the part of the TREWScore paper where the authors describe how they determined which patients met their criteria for infection. Skim the source they cite to find the relevant criteria. Where in the linked paper is the infection criteria found? Which table(s) in MIMIC might we use to assess which patients meet the criteria?

#### 1.12 (5 pts)
The course staff has extracted the entirety of the relevant table from MIMIC and provided it for you in *mystery.csv*.

Additionally, for your convenience, we include the set of reference information from the paper that will be useful in determining which admissions indicate infection. Using this reference information, filter the provided table such that it includes only admissions from the 1000 subject development cohort that have at least one string that *starts with* one of the provided strings that indicate infection.

We suggest using functions from the `stringr` package.


```R
# Provided
infection3digit <- c('001','002','003','004','005','008',
                    '009','010','011','012','013','014','015','016','017','018',
                    '020','021','022','023','024','025','026','027','030','031',
                    '032','033','034','035','036','037','038','039','040','041',
                    '090','091','092','093','094','095','096','097','098','100',
                    '101','102','103','104','110','111','112','114','115','116',
                    '117','118','320','322','324','325','420','421','451','461',
                    '462','463','464','465','481','482','485','486','494','510',
                    '513','540','541','542','566','567','590','597','601','614',
                    '615','616','681','682','683','686','730')
infection4digit <- c('5695','5720','5721','5750','5990','7110',
                    '7907','9966','9985','9993')
infection5digit <- c('49121','56201','56203','56211','56213', '56983')
infection_codes <- c(infection3digit, infection4digit, infection5digit)
```

#### 1.13 (5 pts)
In the paper, the authors also consider a patient to have infection during an admission if there is at least one mention of 'sepsis' or 'septic' in a clinical note for the admission. The course staff has done the work of extracting the clinical notes for the 1000 patients we selected for our development set.

Load the notes data from *notes_small_cohort_v2.csv* into a dataframe. Once you have done so, apply the string matching techniques you developed above to identify admissions that mention the terms 'sepsis' or 'septic'. 

#### 1.14 (5 pts)
At this stage, we now have all the information we need to determine the times that patients meet the criteria for sepsis. Join the results from the search for patients with infection codes and sepsis notes with your SIRS data frame and label the chart times that meet the TREWScore paper's definition of sepsis.

#### 1.15 (2 pts)

In the TREWScore paper, the authors also identify patients with *severe sepsis* and *septic shock*. Severe sepsis is
defined as sepsis with **organ dysfunction**. Septic shock is defined as **severe sepsis**, **hypotension**, and **adequate fluid resuscitation** occurring at the same time.  In order to determine which patients met the criteria for *severe sepsis* and *septic shock* according to the TREWScore paper, we will first need to define the concepts of **organ dysfunction**, **adequate fluid resuscitation**, and **hypotension**.

In order to identify those patients with *severe sepsis*, the authors define *severe sepsis* as patients with sepsis who had sepsis-related organ dysfunction. For their study, the criteria for organ dysfunction are listed in the Supplementary Material. In short, the criteria for organ dysfunction during an admission is at least one of the following:

* Systolic blood pressure (SBP) < 90 mmHg
* Lactate > 2.0 mmol/L
* Urine output < 0.5 mL/kg over the preceding two hours despite adequate fluid resuscitation
* Creatinine > 2.0 mg/dL without the presence of chronic dialysis or renal insufficiency as indicated by an ICD-9 code of V45.11 or 585.9
* Bilirubin > 2 mg/dL without the presence of chronic liver disease and cirrhosis as indicated by an ICD-9 code of 571 and any of the subcodes
* Platelet count < 100,000 μL
* International normalized ratio (INR) > 1.5
* Acute lung injury with PaO2/FiO2 < 200 in the presence of pneumonia indicated by an ICD-9 code of 486
* Acute lung injury with PaO2/FiO2 < 250 in the absence of pneumonia indicated by the absence of an ICD-9 code of 486

Unfortunately, this criteria is rather extensive and tedious to compute and thus we **do not expect you to implement the above set of the criteria.** Instead, we adopt a simpler approach. In the Angus 2001 paper, the authors did just that by defining a set of ICD9 codes as a proxy for sepsis-related organ dysfunction. Look through the Angus paper, determine the criteria, and identify the admissions that meet the criteria for organ dysfunction using one of the dataframes you have already loaded.

**We will only ask you to implement the simplified criteria**.

#### 1.16 (2 pt)
We will now handle with the concept of **adequate fluid resuscitation**. The authors define patients with adequate fluid resuscitation as those whose "Total fluid replacement per kilogram [of bodyweight] over the past 24 hours is ≥20 mL or total fluid replacement is ≥1200 mL". To get the information required to identify those patients who had adequate fluid resuscitation, we need to look to the records in MIMIC III that contain information relevant to infused fluids. Fortunately, the database has tables just for this purpose. Unfortunately, the history of the data collection process is such that this information is split across two similar tables with separate conventions: CareVue and MetaVision. 

First take a look at the overview on how input/output events are stored, located [here](https://mimic.physionet.org/mimicdata/io/). Then, take a closer look at the documentation for the CareVue ([inputevents_cv](https://mimic.physionet.org/mimictables/inputevents_cv/)) and MetaVision ([inputevents_mv](https://mimic.physionet.org/mimictables/inputevents_mv/)) tables, as we will need both to find those patients with adequate fluid resuscitation.

The course staff have queried MIMIC to extract records for the patients in the development set from each of input events tables, *inputevents_cv* and *inputevents_mv*. You can find the relevant definitions for the items in the input event tables in *d_items*. 

We additionally ran [this query](https://github.com/MIT-LCP/mimic-code/blob/bec1470b33352117ba00c62768c94c5ac93d9996/concepts/demographics/HeightWeightQuery.sql) from the MIMIC-Code repository to extract height and weight information for each ICU stay and exported it to *height_weight.csv*. This data will be useful to compute the amount of fluid replacement per kilogram of bodyweight.

Load each of *inputevents_cv_small_cohort.csv*, *inputevents_mv_small_cohort_v2.csv*, *d_items.csv*, and *height_weight.csv* as dataframes. Join the *inputevents_cv* and *inputevents_mv* records with *d_items* to get the relevant item definitions. Inspect the resulting dataframes to get a feel for what data is in each. In a few sentences, describe the major differences between the CareVue and MetaVision inputs.

#### 1.17 (5 pts)

From reading over the documentation, it may appear that we will need to do some work to harmonize the measurement units and rate information in order to correctly assess the amount of fluid infused. However, it is plausible that some simplifying assumptions may be made if we better understand the data. 

In order to get a feel for what sort of data is stored in the input events tables, compute the frequency of each unique combination of *amountuom* (amount unit of measure), *rateuom* (rate unit of measure), and the item *label* in both our input events tables using `dplyr` commands.

#### 1.18 (5 pts)

Recall that the next task we are interested in is obtaining the amount of fluid infused over the past 24 hours, measured in mL.

Unsure of how to proceed with your analysis, you ask a collaborating researcher if they have any suggestions. They propose that you do the following:

1. For both tables, include only those rows with a unit of measurement of mL. 
2. For both tables, ignore rate information.
3. For CareVue data, assume that charttime listed may be considered an endtime for one hour of infusion.
4. For Metavision data, ignore the starttime and consider the whole of the fluid to be infused exactly at the end time.

In 2-4 sentences, respond with whether you think the set of assumptions is valid and discuss any potential biases or errors induced by the assumptions.

#### 1.19 (10 Points)
Implement the suggested strategy for both the CareVue and MetaVision data. Determine whether the patient has adequate fluid resuscitation at each recorded charttime. Once you have done so, join the results obtained from both data sources. You may find it useful to merge columns from the two sources that have the same meaning yet have different names.

**Note**: that there are multiple ways to solve this problem with varying ranges of efficiencies. If you cannot find an efficient solution, we will provide partial credit for a less efficient solution run on a fraction of the patients.

**Hint**: The *zoo* package provides a function that may help you develop an efficient solution.

#### 1.20 (5 pts) 

The authors define *septic shock* as the presence of *severe sepsis* and *hypotension*, where hypotension is defined by systolic blood pressure less than 90 mmHg for at least 30 minutes. Using the timeline you previously developed and any method you like, identify the chart times that correspond to hypotension.

The note and hint from the previous problem additionally applies here.

#### 1.21 (5 pts) Tying it all together
Using the things you have derived so far, devise and implement a strategy to create a timeline that merges the labels you derived for each sepsis grade such that for each unique observation time for each patient, you have a binary label for each of *sepsis*, *severe sepsis*, and *septic shock*. 

Congratulations! You've extracted a patient cohort from MIMIC and derived multiple sepsis-related endpoints. You're done!
