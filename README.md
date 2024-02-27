# discontinuation
////Contraceptive Discontinuation rates
////Prepared by Bridget Waineina
////September -October 2023

clear all

*set directorate
cd "D:\Bridget\Disk d\STATA\School project\take 2"

****************************************************************************
*importing the data
use "D:\Bridget\Disk d\STATA\School project\take 2\KEP3_WealthWeightAll_8Mar2022.dta",clear
*****************************************************************************

*Data cleaning
keep if HHQ_result_cc==1 & FRS_result_cc==1
keep if last_night==1
keep if calendar_events_exist=="1"

*****************************************************************
*Generating age groups

egen Age_Group=cut(FQ_age), at(15(5)50)
label define Age_Group 15 "15-19" 20 "20-24" 25 "25-29" 30 "30-34" 35 "35-39" 40 "40-44" 45 "45-49"
label val Age_Group Age_Group 

*Generating parity
egen parity=cut(birth_events), at(0, 1,2,4, 40) 
label define parity_list 0 "No child" 1 "1 child" 2 "2-3 children" 4 ">4 children", modify
label values parity parity_list

*Marital status
recode FQmarital_status (1 2= 1 "In Union") (3 4 5=2 "Not in Union"), gen (Marital_status)

*Frequency of intercourse per week
replace n_sex_permonth=. if n_sex_permonth==-99| n_sex_permonth==-88
gen n_sex_perweek= n_sex_permonth/4

gen sexperweek=. 
replace sexperweek=1 if n_sex_perweek<2
replace sexperweek=2 if n_sex_perweek>=2 & n_sex_perweek<=35

label define n_sex_perweek 1 "<2/week" 2 ">2/week"
label val sexperweek n_sex_perweek

*Education level

label define education 0 "Never" 1 "Primary" 2 "Post-Primary vocational" 3 "Secondary" 4 "College/ University"

recode school (4 5 =4 "College/ University"),gen (education_level)
label val education_level education

*Unmet need for both spacing and limiting
cap gen unmet_need=0 if unmet !=.
replace unmet_need=1 if unmet_need==0 & unmet ==1 | unmet==2

label define need 1 "unmet need" 0 "met / other"
label val unmet_need need
tab unmet_need,m

*****************************************************************************
*Keeping necessary varialbles
keep member_number FQmetainstanceID current_methodnumEC  Marital_status Age_Group wealthtertile_Kenya FQmarital_status cc_col1_full cc_col2_full ur pill implant IUD injectables education_level doi doi_corrected FQdoi FQdoi_corrected FQdoi_correctedSIF doimonth doiyear doicmc current_user_first current_user recent_user current_userEC recent_userEC totaldemand FQweight* sexperweek parity level1 unmet_need
*************************************************************
*Saving dataset
save "D:\Bridget\Disk d\STATA\School project\take 2\KEP3.dta",replace

use "D:\Bridget\Disk d\STATA\School project\take 2\KEP3.dta",clear

*****************************************************************************
*Checking for duplicates
duplicates tag FQmetainstanceID, gen(d)
keep if d== 0 
drop d
***********************************************************************
*Splitting the columns 

split cc_col1_full, p(,) generate(method_month_)

foreach i of varlist method_month_1-method_month_48{
	replace `i'="81" if `i'=="B"
	replace `i'="82" if `i'=="P"
	replace `i'="83" if `i'=="T"
		}

label define event -99 "No response" 0 "No Method Used" 1 "Female sterilization" 2 "Male sterilization" 3 "Implants" 4 "IUD" 5 "Injectable" 7 "Pill" 8 "Emergency Contraceptives" 9 "Male Condom" 10 "Female Condom" 11 "Diaphragm" 12 "Foam/Jelly" 13 "Standard days/cycle beads" 14 "LAM" 30 "Rhythm" 31 "Withdrawal" 39 "Other Traditional methods" 81 "Birth" 82 "Pregnancy" 83 "Termination",modify
		
foreach var of varlist method_month_1-method_month_48 {
	destring `var', replace
	lab val `var' event
}

*label val method_month event

*Column 2 reason for discontinued
*splitting, labelling and destringing

split cc_col2_full, parse(,) generate (reason_month_)

label define reason 1 "Infrequent_sex/husband_away" 2 "Became pregnant while using" 3 "Wanted_to_be_pregnant" 4 "Husband/Partner_disapproved" 5 "Want_more_effective" 6 "Side_effects_health_concerns" 7 "Lack_access/too_far" 8 "Cost_too_much" 9 "Inconvinience_to_use" 10 "Up_to_god" 11 "Difficult_to_get_pregnant/menopausal" 12 "Marrital_dissolution/Separation" 96 "Other", modify

foreach var of varlist reason_month_1-reason_month_48{
    destring `var', replace
	label val `var' reason
}

***********************************************************************
*Renaming variables following the calendar with m_m_1 being the cc_col1 value of interview data

*Macro for full calendar
global cal_len=36

forvalues i = 1/48 {
   local j = 49 - `i'
    rename method_month_`i' m_m_`j'
}


forvalues i = 1/48 {
   local j = 49 - `i'
    rename reason_month_`i' r_m_`j'
}

*Dropping empty calendar columns

drop m_m_48-m_m_37
drop r_m_48-r_m_37
******************************************************************
local cal_len=36
local cal_len = $cal_len
	
* Set episode number - initialized to 0
gen episodes_tot = 0
* Set previous calendar column 1 variable to anything that won't be in the calendar
gen prev_cal_col = -1

set trace on
* Create variable to identify unique episodes of use
forvalues j = `cal_len'(-1)1 {
  local i = `cal_len' - `j' + 1
  * Increase the episode number if there is a change in cc_col1_
  replace episodes_tot = episodes_tot+1 if m_m_`i' != prev_cal_col
  * Set the episode number
  gen int event_number`i' = episodes_tot
  * Save the cc_col1_* value for the next time through the loop
  replace prev_cal_col = m_m_`i'
}


******************************************************************
use "D:\Bridget\Disk d\STATA\School project\survival\episode_file.dta",clear

////Merging file with attributes and rates and event file

merge 1:1 FQmetainstanceID using "D:\Bridget\Disk d\STATA\School project\take 2\KEP3.dta"

drop if _merge==2
drop _merge

*cleaning
drop member_number doi doi_corrected FQdoi  FQdoi_corrected FQdoi_correctedSIF implant IUD pill injectable

order level1 ur wealthtertile_Kenya  Age_Group parity FQmarital_status Marital_status education_level sexperweek current_methodnumEC ,a(FQmetainstanceID )

****************************************************************
*Saving dataset
save "D:\Bridget\Disk d\STATA\School project\survival\svl_dataset.dta",replace

use "D:\Bridget\Disk d\STATA\School project\survival\svl_dataset.dta",clear

******************************************************************
*Discontinuation episodes across

*******Across reasons***************************
*12m
collapse (count) exposure (percent) per_epi=exposure if exposure<=12 & discont==1, by (reason)

*24m
collapse (count) exposure (percent) per_epi=exposure if exposure<=24 & discont==1, by (reason)

*36m
collapse (count) exposure (percent) per_epi=exposure if exposure<=36 & discont==1, by (reason)

******Across methods***************************
*12m
collapse (count) exposure (percent) per_epi=exposure if exposure<=12 & discont==1, by (method)

*24m
collapse (count) exposure (percent) per_epi=exposure if exposure<=24 & discont==1, by (method)

*36m
collapse (count) exposure (percent) per_epi=exposure if exposure<=36 & discont==1, by (method)

********Across age groups*************************
*12m
collapse (count) exposure (percent) per_epi=exposure if exposure<=12 & discont==1, by (Age_Group)

*24m
collapse (count) exposure (percent) per_epi=exposure if exposure<=24 & discont==1,  by (Age_Group)
*36m
collapse (count) exposure (percent) per_epi=exposure if exposure<=36 & discont==1,  by (Age_Group)

**************Across UR**********************
*12m
collapse (count) exposure (percent) per_epi=exposure if exposure<=12 & discont==1, by (ur)

*24m
collapse (count) exposure (percent) per_epi=exposure if exposure<=24 & discont==1,  by (ur)
*36m
collapse (count) exposure (percent) per_epi=exposure if exposure<=36 & discont==1,  by (ur)

*****************Across education level*****************
*12m
collapse (count) exposure (percent) per_epi=exposure if exposure<=12 & discont==1, by (education_level)

*24m
collapse (count) exposure (percent) per_epi=exposure if exposure<=24 & discont==1, by (education_level)
*36m
collapse (count) exposure (percent) per_epi=exposure if exposure<=36 & discont==1, by (education_level)

****************Across marital status********************
*12m
collapse (count) exposure (percent) per_epi=exposure if exposure<=12 & discont==1, by (Marital_status)

*24m
collapse (count) exposure (percent) per_epi=exposure if exposure<=24 & discont==1, by (Marital_status)
*36m
collapse (count) exposure (percent) per_epi=exposure if exposure<=36 & discont==1, by (Marital_status)

********************Across wealth tertiles************************
*12m
collapse (count) exposure (percent) per_epi=exposure if exposure<=12 & discont==1, by (wealthtertile_Kenya)

*24m
collapse (count) exposure (percent) per_epi=exposure if exposure<=24 & discont==1, by (wealthtertile_Kenya)
*36m
collapse (count) exposure (percent) per_epi=exposure if exposure<=36 & discont==1, by (wealthtertile_Kenya)

********************Across urbanisation*********************************
*12m
collapse (count) exposure (percent) per_epi=exposure if exposure<=12 & discont==1, by (ur)

*24m
collapse (count) exposure (percent) per_epi=exposure if exposure<=24 & discont==1, by (ur)
*36m
collapse (count) exposure (percent) per_epi=exposure if exposure<=36 & discont==1, by (ur)

****************************************************************

****SURVIVAL ANALYSIS
*Model
use "D:\Bridget\Disk d\STATA\School project\survival\svl_dataset.dta",clear

stset exposure, failure(discont==1)

*univariate models
stcox i.method
stcox b4.education_level
stcox i.Age_Group
stcox b3.wealthtertile_Kenya
stcox b2.ur
stcox i.Marital_status
stcox i.county

*Multivariate (backward)
stcox i.method i.Age_Group i.ur i.Marital_status i.education_level i.wealthtertile_Kenya i.county

stcox method Age_Group ur Marital_status education_level wealthtertile_Kenya county

*Testing
test  2.wealthtertile_Kenya 3.wealthtertile_Kenya ///insignificant

test 1.education_level 2.education_level 3.education_level 4.education_level ///insignificant

test 20.Age_Group 25.Age_Group 30.Age_Group 35.Age_Group 40.Age_Group 45.Age_Group //significant

test 2.method 3.method 4.method 5.method 7.method 8.method 9.method 14.method //significant

stcox i.method i.Age_Group i.ur i.Marital_status i.education_level i.wealthtertile_Kenya i.county
estat ic //  12654.41

stcox i.method i.Age_Group i.ur  i.education_level i.wealthtertile_Kenya i.county
*wealth insignificant 
estat ic //12657.88 

stcox i.method i.Age_Group i.ur  i.education_level i.county
*education insignificant

stcox i.method i.Age_Group i.ur i.county
test 2.ur //significant
test 2.county 3.county 4.county 5.county 6.county 7.county 8.county 9.county 10.county 11.county

stcox i.method i.Age_Group i.ur i.county
estat ic // 15402.35

stcox i.method i.Age_Group
estat ic // 15399.66

stcox method Age_Group

***Interarctions

stcox i.method i.Age_Group c.method#i.Age_Group  //sig

stcox i.method i.Age_Group c.method#c.Age_Group
*p-value 0.678 : insignificant 

stcox i.method i.Age_Group i.ur i.county c.method#i.ur //insignificant

stcox i.method i.Age_Group i.ur i.county c.Age_Group#i.ur //insignificant

stcox i.method i.Age_Group i.ur i.county c.method#c.county //isignificant

stcox i.method i.Age_Group i.ur i.county c.Age_Group#c.county //insignificant

stcox i.method i.Age_Group i.ur i.county c.ur#c.county //insignificant

stcox i.method i.Age_Group c.method#c.Age_Group //no interactions

//All tested Interactions are insignificant


stcox i.method i.Age_Group 
estat ic // 15399.66

stcox i.method i.Age_Group i.ur i.county

estat ic // 12651.21

**Goodness of fit

stcox i.method i.Age_Group
estimate store model1
stcox i.method i.Age_Group i.ur i.county
estimates store model2
lrtest model1 model2

*Model 1 has a better fit

clear all
