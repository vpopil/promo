---
title: DataGrab
notebook: what.ipynb
nav_include: 4
---

## Contents
{:.no_toc}
*  
{: toc}



# Crime EDA





## Scraping FBI data



```python
# create array of years
years = np.arange(2006,2017,1)
```




```python
def read_fbi_year_data(year):
    
    # dataframe to store the data
    df = pd.DataFrame()

    # get url of webpage with MSA data
    if year < 2010:
        url = "https://www2.fbi.gov/ucr/cius" + str(year) + "/data/table_06.html"
    elif year in [2012,2013] :
        url = "https://ucr.fbi.gov/crime-in-the-u.s/"+ str(year) +"/crime-in-the-u.s.-"\
            + str(year) + "/tables/6tabledatadecpdf/table-6"
    elif year < 2016:
        url = "https://ucr.fbi.gov/crime-in-the-u.s/"\
            + str(year) + "/crime-in-the-u.s.-" + str(year) + "/tables/table-6"
    else: 
        url = "https://ucr.fbi.gov/crime-in-the-u.s/"+ str(year) +"/crime-in-the-u.s.-"\
            + str(year) +"/topic-pages/tables/table-4"
    
    # get HTML from the page and check the response code           
    response = requests.get(url) 
         
    # proceed if 200
    if response.ok:
        # create instance of BeautifulSoup from the response text
        soup = BeautifulSoup(response.text,"html.parser")

        # fetch the first table matching class criteria
        table = soup.find("table", {"class":"data"})
        
        # get the table body
        tbody = table.find("tbody")

        # fetch all rows from the table
        rows = tbody.find_all("tr")

        # create list to store MSAs
        msa_murders = []
        
        # create dictionary to store MSAs and rates
        d = {}

        # iterate over rows
        for row in rows:
        
            # get header line (MSA)
            header_line = row.find("th", {"class":"subguide1"})\
                or row.find("th", {"class":"subguide2"})\
                or row.find("th", {"class":"even group0 alignleft valignmenttop"})\
                or row.find("th", {"class":"even group0 bold alignleft valignmenttop"})\
                or row.find("th", {"class":"even group0 bold valignmenttop"})

            # store MSA if found    
            if header_line:
                msa = header_line.text
                msa = msa.replace("","-")
                msa = msa.replace("\n"," ")
                msa = msa.strip()
                msa = msa[:msa.find(" M.S.A.")]
                
                # update dict
                d.update({"MSA":msa})
            else:
                # var to store murder rate
                murder_per_100_k = 0

                # get the table entry
                line = row.find("th", {"class":"subguide1a"})\
                    or row.find("th", {"class":"subguide1e"})\
                    or row.find("th", {"class":"odd group1 alignleft valignmentbottom"})\
                    or row.find("th", {"class":"odd group1 valignmentbottom"})
                                                
                line_label = ""
                
                if line:
                    line_label = line.text

                # if match the criteria, store rate
                if line_label.strip() == "Rate per 100,000 inhabitants":
                    # set custom index position (2007 is exception)
                    index = 2 if year != 2007 else 1
                    
                    # get murder rate
                    murder_per_100_k = row.find_all("td")[index].text.strip("\n")

                    # update dict, append to list and refresh dictionary
                    d.update({"murder_per_100_k":murder_per_100_k})
                    msa_murders.append(d)
                    d = {}
                    
        # create dataframe and drop nan (caused by sub-msa's)
        df = pd.DataFrame(msa_murders)
        df = df.dropna()
    return df
```




```python
# create dictionary to store dataframes with census data
fbi_years_dict = {}

# iterate over years and read the data, storing into dict
for year in years:
    # read and add to dict
    fbi_years_dict.update({year:read_fbi_year_data(year)})
    time.sleep(1)
```




```python
# take a peak at 2010 data
fbi_years_dict[2010].head()
```





<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>MSA</th>
      <th>murder_per_100_k</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Abilene, TX</td>
      <td>3.1</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Akron, OH</td>
      <td>3.7</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Albany, GA</td>
      <td>8.7</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Albany-Schenectady-Troy, NY</td>
      <td>1.5</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Albuquerque, NM</td>
      <td>5.8</td>
    </tr>
  </tbody>
</table>
</div>



## Read Census Data



```python
def read_census_year_data(year):
    # parse last two digits of year
    year_last_two = str(year)[-2:]
    
    # custom suffix of data (2006 exception)
    suffix = '_EST' if year == 2006 else '_1YR'
    
    # set path and name of data files
    path = "../data/census/raw/ACS_" + year_last_two + suffix +"_S0201/"
    file = "ACS_" + year_last_two + suffix + "_S0201.csv"

    # read data
    df = pd.read_csv(path + file, header=0, dtype=object)
    
    # add year column to the data
    df['year'] = year
    
    # remove suffix in the MSA name
    df['GEO.display-label'] = df['GEO.display-label'].apply(lambda x: x[:x.find(" Metro Area")])

    # get only subset of columns (avoid MOE columns)
    cols = ['year','GEO.display-label'] + [c for c in df.columns if c[:3] == 'EST']
    
    # keep only data for all races (total)
    df = df[df['POPGROUP.id'] == "001"]
   
    # get filtered columns
    df=df[cols]
    
    return df
```




```python
# create dictionary to store dataframes with census data
census_years_dict = {}

# iterate over years and read the data, storing into dict
for year in years:
    # read and add to dict
    census_years_dict.update({year:read_census_year_data(year)})
```


### Process 2010 data for EDA

_This is done only for 2010 year for EDA, once features subset is decided, further processing will be done for all datasets._



```python
# get a copy of data
eda_df = census_years_dict[2010].copy()

# rename columns for EDA for 2010 year
col_rename_map_2010 = { 'GEO.display-label':'MSA',
                        'EST_VC11':'total_population',
                        'EST_VC12':'gender_male',
                        'EST_VC13':'gender_female',
                        'EST_VC15':'age_under_5_years',
                        'EST_VC16':'age_5_to_17_years',
                        'EST_VC17':'age_18_to_24_years',
                        'EST_VC18':'age_25_to_34_years',
                        'EST_VC19':'age_35_to_44_years',
                        'EST_VC20':'age_45_to_54_years',
                        'EST_VC21':'age_55_to_64_years',
                        'EST_VC22':'age_65_to_74_years',
                        'EST_VC23':'age_75_years_and_over',
                        'EST_VC25':'age_median_age_(years)',
                        'EST_VC27':'age_18_years_and_over',
                        'EST_VC28':'age_21_years_and_over',
                        'EST_VC29':'age_62_years_and_over',
                        'EST_VC30':'age_65_years_and_over',
                        'EST_VC55':'population_in_households_householder_or_spouse',
                        'EST_VC56':'population_in_households_child',
                        'EST_VC58':'population_in_households_nonrelatives',
                        'EST_VC59':'population_in_households_nonrelatives_unmarried_partner',
                        'EST_VC64':'family_households',
                        'EST_VC66':'family_households_with_own_children_under_18_years',
                        'EST_VC67':'family_households_married-couple_family',
                        'EST_VC68':'family_household_married_couple_family_with_own_children_under_18_years',
                        'EST_VC69':'family_households_female_householder_no_husband_present',
                        'EST_VC70':'family_households_female_householder_no_husband_present_with_own_children_under_18',
                        'EST_VC71':'nonfamily_households',
                        'EST_VC72':'nonfamily_households_male_householder',
                        'EST_VC73':'nonfamily_households_male_householder_living_alone',
                        'EST_VC74':'nonfamily_households_male_householder_not_living_alone',
                        'EST_VC75':'nonfamily_households_female_householder',
                        'EST_VC76':'nonfamily_households_female_householder_living_alone',
                        'EST_VC77':'nonfamily_households_female_householder_not_living_alone',
                        'EST_VC79':'average_household_size',
                        'EST_VC80':'average_family_size',
                        'EST_VC85':'now_married_except_separated',
                        'EST_VC86':'widowed',
                        'EST_VC87':'divorced',
                        'EST_VC88':'separated',
                        'EST_VC89':'never_married',
                        'EST_VC108':'enrolled_nursery_school_or_preschool',
                        'EST_VC109':'enrolled_kindergarten',
                        'EST_VC110':'enrolled_elementary_school_grades_1_8',
                        'EST_VC111':'enrolled_high_school_grades_9_12',
                        'EST_VC112':'enrolled_college_or_graduate_school',
                        'EST_VC124':'less_than_high_school_diploma',
                        'EST_VC125':'high_school_graduate_(includes_equivalency)',
                        'EST_VC126':'some_college_or_associates_degree',
                        'EST_VC127':'bachelors_degree',
                        'EST_VC128':'graduate_or_professional_degree',
                        'EST_VC130':'high_school_graduate_or_higher',
                        'EST_VC133':'bachelors_degree_or_higher',
                        'EST_VC142':'unmarried_portion_of_women_15_to_50_years_who_had_a_birth_in_past_12_months',
                        'EST_VC147':'population_30_years_and_over_living_with_grandchild(ren)',
                        'EST_VC148':'population_30_years_and_over_living_with_grandchild(ren)_responsible_for_grandchild(ren)',
                        'EST_VC153':'civilian_veteran',
                        'EST_VC158':'total_civilian_noninst_population_with_a_disability',
                        'EST_VC161':'civilian_noninst_population_under_18_years_with_a_disability',
                        'EST_VC164':'civilian_noninst_population_18_to_64_years_with_a_disability',
                        'EST_VC167':'civilian_noninst_population_65_years_and_older_with_a_disability',                   
                        'EST_VC172':'residence_year_ago_same_house',                    
                        'EST_VC173':'residence_year_ago_different_house_in_the_us',
                        'EST_VC174':'residence_year_ago_different_house_in_the_us_same_county',
                        'EST_VC175':'residence_year_ago_different_house_in_the_us_different_county',
                        'EST_VC176':'residence_year_ago_different_house_in_the_us_different_county_same_state',
                        'EST_VC177':'residence_year_ago_different_house_in_the_us_different_county_different_state',
                        'EST_VC178':'residence_year_ago_abroad',
                        'EST_VC182':'native',   
                        'EST_VC186':'foreign_born',
                        'EST_VC190':'foreign_born_naturalized_us_citizen',
                        'EST_VC194':'foreign_born_not_a_us_citizen',
                        'EST_VC199':'born_outside_entered_2000_or_later',
                        'EST_VC200':'born_outside_entered_1990_to_1999',
                        'EST_VC201':'born_outside_entered_before_1990',
                        'EST_VC206':'born_in_europe',
                        'EST_VC207':'born_in_asia',
                        'EST_VC208':'born_in_africa',
                        'EST_VC209':'born_in_oceania',
                        'EST_VC210':'born_in_latin_america',
                        'EST_VC211':'born_in_northern_america',
                        'EST_VC216':'speaking_english_only',
                        'EST_VC217':'speaking_language_other_than_english',
                        'EST_VC218':'speaking_language_other_than_english_speak_english_less_than_very_well',
                        'EST_VC223':'employment_in_labor_force',    
                        'EST_VC224':'employment_in_labor_force_civilian_labor_force',
                        'EST_VC225':'employment_in_labor_force_civilian_labor_force_employed',
                        'EST_VC226':'employment_in_labor_force_civilian_labor_force_unemployed',
                        'EST_VC227':'employment_in_labor_force_civilian_labor_force_unemployed_percent_of_civilian_labor_force',
                        'EST_VC228':'employment_in_labor_force_armed_forces',
                        'EST_VC229':'employment_not_in_labor_force',  
                        'EST_VC241':'commuting_car_truck_or_van_drove_alone',
                        'EST_VC242':'commuting_car_truck_or_van_carpooled',
                        'EST_VC243':'commuting_public_transportation_(excluding_taxicab)',
                        'EST_VC244':'commuting_walked',
                        'EST_VC245':'commuting_other_means',
                        'EST_VC246':'commuting_worked_at_home',
                        'EST_VC247':'commuting_mean_travel_time_to_work_(minutes)',
                        'EST_VC252':'occupation_management,_business,_science,_and_arts_occupations',
                        'EST_VC253':'occupation_service_occupations',
                        'EST_VC254':'occupation_sales_and_office_occupations',
                        'EST_VC255':'occupation_natural_resources_construction_and_maintenance_occupations',
                        'EST_VC256':'occupation_production_transportation_and_material_moving_occupations',
                        'EST_VC275':'industry_agriculture_forestry_fishing_and_hunting_and_mining',
                        'EST_VC276':'industry_construction',
                        'EST_VC277':'industry_manufacturing',
                        'EST_VC278':'industry_wholesale_trade',
                        'EST_VC279':'industry_retail_trade',
                        'EST_VC280':'industry_transportation_and_warehousing_and_utilities',
                        'EST_VC281':'industry_information',
                        'EST_VC282':'industry_finance_and_insurance_and_real_estate_and_rental_and_leasing',
                        'EST_VC283':'industry_professional_scientific_and_management_and_administrative_and_waste_management_services',
                        'EST_VC284':'industry_educational_services_and_health_care_and_social_assistance',
                        'EST_VC285':'industry_arts_entertainment_and_recreation_and_accommodation_and_food_services',
                        'EST_VC286':'industry_other_services_(except_public_administration)',
                        'EST_VC287':'industry_public_administration',
                        'EST_VC292':'private_wage_and_salary_workers',
                        'EST_VC293':'government_workers',
                        'EST_VC294':'self_employed_workers_in_own_not_incorporated_business',
                        'EST_VC295':'unpaid_family_workers',
                        'EST_VC300':'median_household_income_(dollars)',
                        'EST_VC302':'households_with_earnings',
                        'EST_VC304':'households__with_social_security_income',
                        'EST_VC306':'households_with_supplemental_security_income',
                        'EST_VC308':'households_with_cash_public_assistance_income',
                        'EST_VC310':'households_with_retirement_income',
                        'EST_VC312':'households_with_food_stamp_snap_benefits',
                        'EST_VC315':'median_family_income_(dollars)',
                        'EST_VC316':'percentage_married-couple_family',
                        'EST_VC317':'median_family_income_(dollars)_married-couple_family',
                        'EST_VC318':'percentage_male_householder_no_spouse_present_family',
                        'EST_VC319':'median_family_income_(dollars)_male_householder_no_spouse_present_family',
                        'EST_VC320':'percentage_female_householder_no_husband_present_family',
                        'EST_VC321':'median_family_income_(dollars)_female_householder_no_husband_present_family',
                        'EST_VC324':'per_capita_income_(dollars)',
                        'EST_VC340':'civilian_noninst_population_with_private_health_insurance',
                        'EST_VC341':'civilian_noninst_population_with_public_health_coverage',
                        'EST_VC342':'civilian_noninst_population_no_health_insurance_coverage',
                        'EST_VC345':'poverty_all_families',
                        'EST_VC346':'poverty_all_families_with_related_children_under_18_years',
                        'EST_VC347':'poverty_all_families_with_related_children_under_18_years_with_related_children_under_5_years_only',
                        'EST_VC348':'poverty_married-couple_family',
                        'EST_VC349':'poverty_married-couple_family_with_related_children_under_18_years',
                        'EST_VC350':'poverty_married-couple_family_with_related_children_under_5_years_only',
                        'EST_VC351':'poverty_female_householder_no_husband_present',
                        'EST_VC352':'poverty_female_householder_no_husband_present_with_related_children_under_18_years',
                        'EST_VC353':'poverty_female_householder_no_husband_present_with_related_children_under_5_years_only',
                        'EST_VC355':'poverty_all_people',
                        'EST_VC356':'poverty_under_18_years',
                        'EST_VC357':'poverty_related_children_under_18_years',
                        'EST_VC358':'poverty_related_children_under_5_years',
                        'EST_VC359':'poverty_related_children_5_to_17_years',
                        'EST_VC360':'poverty_18_years_and_over',
                        'EST_VC361':'poverty_18_to_64_years',
                        'EST_VC362':'poverty_65_years_and_over',
                        'EST_VC363':'poverty_people_in_families',
                        'EST_VC364':'poverty_unrelated_individuals_15_years_and_over',
                        'EST_VC369':'owner_occupied_housing_units',
                        'EST_VC370':'renter_occupied_housing_units',
                        'EST_VC372':'average_household_size_of_owner-occupied_unit',
                        'EST_VC373':'average_household_size_of_renter-occupied_unit',
                        'EST_VC378':'units_in_structure_1_unit_detached_or_attached',
                        'EST_VC379':'units_in_structure_2_to_4_units',
                        'EST_VC380':'units_in_structure_5_or_more_units',
                        'EST_VC381':'units_in_structure_mobile_home_boat_rv_van_etc',
                        'EST_VC386':'housing_unit_built_2000_or_later',
                        'EST_VC387':'housing_unit_built_1990_to_1999',
                        'EST_VC388':'housing_unit_built_1980_to_1989',
                        'EST_VC389':'housing_unit_built_1960_to_1979',
                        'EST_VC390':'housing_unit_built_1940_to_1959',
                        'EST_VC391':'housing_unit_built_1939_or_earlier',
                        'EST_VC396':'vehicles_per_housing_unit_none',
                        'EST_VC397':'vehicles_per_housing_unit_1_or_more',
                        'EST_VC402':'house_heating_fuel_gas',
                        'EST_VC403':'house_heating_fuel_electricity',
                        'EST_VC404':'house_heating_fuel_all_other_fuels',
                        'EST_VC405':'house_heating_fuel_no_fuel_used',
                        'EST_VC409':'occupied_housing_units',
                        'EST_VC410':'no_telephone_service_available',
                        'EST_VC411':'1_01_or_more_occupants_per_room',
                        'EST_VC416':'monthly_owner_costs_as_percentage_of_household_income_less_than_30_percent',
                        'EST_VC417':'monthly_owner_costs_as_percentage_of_household_income_30_percent_or_more',
                        'EST_VC422':'house_median_value_(dollars)',
                        'EST_VC423':'house_median_selected_monthly_owner_costs_with_a_mortgage_(dollars)',
                        'EST_VC424':'house_median_selected_monthly_owner_costs_without_a_mortgage_(dollars)',
                        'EST_VC429':'gross_rent_as_percentage_of_household_income_less_than_30_percent',
                        'EST_VC430':'gross_rent_as_percentage_of_household_income_30_percent_or_more',
                        'EST_VC435':'median_gross_rent_(dollars)'}

# rename the columns
eda_df = eda_df.rename(columns=col_rename_map_2010)

# get list of columns to retain
cols_to_keep = [c for c in eda_df.columns if c[:3] != 'EST']

# update columns
eda_df = eda_df[cols_to_keep]

# take a peak at df
eda_df.head()
```





<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>year</th>
      <th>MSA</th>
      <th>total_population</th>
      <th>gender_male</th>
      <th>gender_female</th>
      <th>age_under_5_years</th>
      <th>age_5_to_17_years</th>
      <th>age_18_to_24_years</th>
      <th>age_25_to_34_years</th>
      <th>age_35_to_44_years</th>
      <th>...</th>
      <th>no_telephone_service_available</th>
      <th>1_01_or_more_occupants_per_room</th>
      <th>monthly_owner_costs_as_percentage_of_household_income_less_than_30_percent</th>
      <th>monthly_owner_costs_as_percentage_of_household_income_30_percent_or_more</th>
      <th>house_median_value_(dollars)</th>
      <th>house_median_selected_monthly_owner_costs_with_a_mortgage_(dollars)</th>
      <th>house_median_selected_monthly_owner_costs_without_a_mortgage_(dollars)</th>
      <th>gross_rent_as_percentage_of_household_income_less_than_30_percent</th>
      <th>gross_rent_as_percentage_of_household_income_30_percent_or_more</th>
      <th>median_gross_rent_(dollars)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1</th>
      <td>2010</td>
      <td>Akron, OH</td>
      <td>702951</td>
      <td>48.6</td>
      <td>51.4</td>
      <td>5.6</td>
      <td>16.7</td>
      <td>10.6</td>
      <td>11.7</td>
      <td>12.5</td>
      <td>...</td>
      <td>6.2</td>
      <td>1.0</td>
      <td>69.3</td>
      <td>30.7</td>
      <td>145000</td>
      <td>1279</td>
      <td>450</td>
      <td>48.0</td>
      <td>52.0</td>
      <td>702</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2010</td>
      <td>Albany-Schenectady-Troy, NY</td>
      <td>870832</td>
      <td>48.9</td>
      <td>51.1</td>
      <td>5.4</td>
      <td>15.9</td>
      <td>11.1</td>
      <td>12.1</td>
      <td>13.1</td>
      <td>...</td>
      <td>2.0</td>
      <td>1.5</td>
      <td>69.7</td>
      <td>30.3</td>
      <td>199000</td>
      <td>1605</td>
      <td>558</td>
      <td>49.8</td>
      <td>50.2</td>
      <td>846</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2010</td>
      <td>Albuquerque, NM</td>
      <td>892014</td>
      <td>49.0</td>
      <td>51.0</td>
      <td>6.8</td>
      <td>17.6</td>
      <td>9.8</td>
      <td>13.8</td>
      <td>12.9</td>
      <td>...</td>
      <td>3.1</td>
      <td>2.8</td>
      <td>63.7</td>
      <td>36.3</td>
      <td>183300</td>
      <td>1305</td>
      <td>358</td>
      <td>50.9</td>
      <td>49.1</td>
      <td>748</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2010</td>
      <td>Allentown-Bethlehem-Easton, PA-NJ</td>
      <td>822141</td>
      <td>48.8</td>
      <td>51.2</td>
      <td>5.7</td>
      <td>17.1</td>
      <td>8.8</td>
      <td>11.2</td>
      <td>13.4</td>
      <td>...</td>
      <td>1.3</td>
      <td>1.2</td>
      <td>63.2</td>
      <td>36.8</td>
      <td>218700</td>
      <td>1653</td>
      <td>555</td>
      <td>44.6</td>
      <td>55.4</td>
      <td>848</td>
    </tr>
    <tr>
      <th>5</th>
      <td>2010</td>
      <td>Atlanta-Sandy Springs-Marietta, GA</td>
      <td>5288302</td>
      <td>48.7</td>
      <td>51.3</td>
      <td>7.2</td>
      <td>19.3</td>
      <td>9.2</td>
      <td>14.5</td>
      <td>15.6</td>
      <td>...</td>
      <td>2.4</td>
      <td>2.8</td>
      <td>60.7</td>
      <td>39.3</td>
      <td>175900</td>
      <td>1544</td>
      <td>448</td>
      <td>46.3</td>
      <td>53.7</td>
      <td>910</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 190 columns</p>
</div>





```python
# join FBI and Census data (ATTENTION, SOME MSA's data will be lost due to unmatch, but acceptable)
eda_df = pd.merge(eda_df,fbi_years_dict[2010], on=['MSA'])
```




```python
# export data to CSV and pickle
eda_df.to_csv("../data/merged/eda_2010.csv", sep=',',index=False)
eda_df.to_pickle("../data/merged/eda_2010.pkl")
```


### Process All Data And Combine into One Dataframe



```python
# create dictionary of selected feature names (easier to rename if needed)
census_features_dict = {
    'feature_1':'now_married_except_separated',
    'feature_2':'less_than_high_school_diploma',
    'feature_3':'unmarried_portion_of_women_15_to_50_years_who_had_a_birth_in_past_12_months',
    'feature_4':'households_with_food_stamp_snap_benefits',
    'feature_5':'percentage_married-couple_family',
    'feature_6':'percentage_female_householder_no_husband_present_family',
    'feature_7':'poverty_all_people',
    'feature_8':'house_median_value_(dollars)'
}

# create dictionary to store mapping dictionaries of feature codes for each year, and update the dictionary
cols_mapping_dicts = {}

cols_mapping_dicts[2006] = { 
    'GEO.display-label':'MSA',
    'EST_VC60':census_features_dict['feature_1'],
    'EST_VC92':census_features_dict['feature_2'],
    'EST_VC107':census_features_dict['feature_3'],
    'EST_VC242':census_features_dict['feature_4'],
    'EST_VC245':census_features_dict['feature_5'],
    'EST_VC249':census_features_dict['feature_6'],
    'EST_VC272':census_features_dict['feature_7'],
    'EST_VC316':census_features_dict['feature_8']}
    
cols_mapping_dicts[2007] = { 
    'GEO.display-label':'MSA',
    'EST_VC65':census_features_dict['feature_1'],
    'EST_VC97':census_features_dict['feature_2'],
    'EST_VC112':census_features_dict['feature_3'],
    'EST_VC247':census_features_dict['feature_4'],
    'EST_VC250':census_features_dict['feature_5'],
    'EST_VC254':census_features_dict['feature_6'],
    'EST_VC277':census_features_dict['feature_7'],
    'EST_VC321':census_features_dict['feature_8']}

cols_mapping_dicts[2008] = { 
    'GEO.display-label':'MSA',
    'EST_VC66':census_features_dict['feature_1'],
    'EST_VC98':census_features_dict['feature_2'],
    'EST_VC113':census_features_dict['feature_3'],
    'EST_VC249':census_features_dict['feature_4'],
    'EST_VC252':census_features_dict['feature_5'],
    'EST_VC256':census_features_dict['feature_6'],
    'EST_VC279':census_features_dict['feature_7'],
    'EST_VC329':census_features_dict['feature_8']}

cols_mapping_dicts[2009] = { 
    'GEO.display-label':'MSA',
    'EST_VC66':census_features_dict['feature_1'],
    'EST_VC98':census_features_dict['feature_2'],
    'EST_VC113':census_features_dict['feature_3'],
    'EST_VC249':census_features_dict['feature_4'],
    'EST_VC252':census_features_dict['feature_5'],
    'EST_VC256':census_features_dict['feature_6'],
    'EST_VC284':census_features_dict['feature_7'],
    'EST_VC334':census_features_dict['feature_8']}

cols_mapping_dicts[2010] = { 
    'GEO.display-label':'MSA',
    'EST_VC85':census_features_dict['feature_1'],
    'EST_VC124':census_features_dict['feature_2'],
    'EST_VC142':census_features_dict['feature_3'],
    'EST_VC312':census_features_dict['feature_4'],
    'EST_VC316':census_features_dict['feature_5'],
    'EST_VC320':census_features_dict['feature_6'],
    'EST_VC355':census_features_dict['feature_7'],
    'EST_VC422':census_features_dict['feature_8']}

# here the mappings for the years are the same
cols_mapping_dicts[2011] = cols_mapping_dicts[2010]
cols_mapping_dicts[2012] = cols_mapping_dicts[2011]

cols_mapping_dicts[2013] = { 
    'GEO.display-label':'MSA',
    'EST_VC93':census_features_dict['feature_1'],
    'EST_VC135':census_features_dict['feature_2'],
    'EST_VC154':census_features_dict['feature_3'],
    'EST_VC332':census_features_dict['feature_4'],
    'EST_VC336':census_features_dict['feature_5'],
    'EST_VC340':census_features_dict['feature_6'],
    'EST_VC376':census_features_dict['feature_7'],
    'EST_VC444':census_features_dict['feature_8']}

# here the mappings for the years are the same
cols_mapping_dicts[2014] = cols_mapping_dicts[2013]

cols_mapping_dicts[2015] = { 
    'GEO.display-label':'MSA',
    'EST_VC93':census_features_dict['feature_1'],
    'EST_VC135':census_features_dict['feature_2'],
    'EST_VC154':census_features_dict['feature_3'],
    'EST_VC332':census_features_dict['feature_4'],
    'EST_VC336':census_features_dict['feature_5'],
    'EST_VC340':census_features_dict['feature_6'],
    'EST_VC376':census_features_dict['feature_7'],
    'EST_VC445':census_features_dict['feature_8']}   

# here the mappings for the years are the same
cols_mapping_dicts[2016] = cols_mapping_dicts[2015]
```




```python
#Function to select features specified, map the names and return the df
def get_selected_features(df, mapping_dict):
    # get a copy of data
    df = df.copy()
    
    # rename the columns and get only those
    df = df.rename(columns=mapping_dict)
    df = df[['year']+list(mapping_dict.values())]

    return df
```




```python
# create df to store all data
all_df = pd.DataFrame()

# repeat for each year adding joined data into one datafreame
for year in years:

    # get census processed data and merge to fbi data
    year_df = pd.merge(get_selected_features(census_years_dict[year], cols_mapping_dicts[year]),
                       fbi_years_dict[year],
                       on=['MSA'])
    
    # add data for this year into the combined dataframe
    all_df = pd.concat([all_df, year_df])

# reset index
all_df = all_df.reset_index(drop=True)

# set datatypes
all_df['year'] = all_df['year'].astype(int)
all_df['murder_per_100_k'] = all_df['murder_per_100_k'].astype(float)
all_df[census_features_dict['feature_1']] = all_df[census_features_dict['feature_1']].astype(float)
all_df[census_features_dict['feature_2']] = all_df[census_features_dict['feature_2']].astype(float)
all_df[census_features_dict['feature_3']] = all_df[census_features_dict['feature_3']].astype(float)
all_df[census_features_dict['feature_4']] = all_df[census_features_dict['feature_4']].astype(float)
all_df[census_features_dict['feature_5']] = all_df[census_features_dict['feature_5']].astype(float)
all_df[census_features_dict['feature_6']] = all_df[census_features_dict['feature_6']].astype(float)
all_df[census_features_dict['feature_7']] = all_df[census_features_dict['feature_7']].astype(float)
all_df[census_features_dict['feature_8']] = all_df[census_features_dict['feature_8']].astype(int)                                                   

all_df.head()
```





<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>year</th>
      <th>MSA</th>
      <th>now_married_except_separated</th>
      <th>less_than_high_school_diploma</th>
      <th>unmarried_portion_of_women_15_to_50_years_who_had_a_birth_in_past_12_months</th>
      <th>households_with_food_stamp_snap_benefits</th>
      <th>percentage_married-couple_family</th>
      <th>percentage_female_householder_no_husband_present_family</th>
      <th>poverty_all_people</th>
      <th>house_median_value_(dollars)</th>
      <th>murder_per_100_k</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2006</td>
      <td>Atlanta-Sandy Springs-Marietta, GA</td>
      <td>49.2</td>
      <td>14.2</td>
      <td>34.3</td>
      <td>5.9</td>
      <td>72.6</td>
      <td>20.0</td>
      <td>11.9</td>
      <td>186800</td>
      <td>7.4</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2006</td>
      <td>Austin-Round Rock, TX</td>
      <td>48.7</td>
      <td>13.7</td>
      <td>30.9</td>
      <td>5.9</td>
      <td>75.0</td>
      <td>17.0</td>
      <td>13.0</td>
      <td>164100</td>
      <td>1.9</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2006</td>
      <td>Baltimore-Towson, MD</td>
      <td>47.2</td>
      <td>14.0</td>
      <td>31.4</td>
      <td>5.5</td>
      <td>71.8</td>
      <td>21.4</td>
      <td>9.0</td>
      <td>300600</td>
      <td>13.3</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2006</td>
      <td>Birmingham-Hoover, AL</td>
      <td>50.9</td>
      <td>15.8</td>
      <td>34.7</td>
      <td>7.1</td>
      <td>72.5</td>
      <td>21.6</td>
      <td>13.1</td>
      <td>131400</td>
      <td>12.5</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2006</td>
      <td>Buffalo-Niagara Falls, NY</td>
      <td>47.1</td>
      <td>12.9</td>
      <td>38.0</td>
      <td>9.6</td>
      <td>72.8</td>
      <td>20.9</td>
      <td>14.2</td>
      <td>105000</td>
      <td>7.5</td>
    </tr>
  </tbody>
</table>
</div>





```python
msa_orig_map_to_corr = {'Atlanta-Sandy Springs-Roswell, GA':'Atlanta-Sandy Springs-Marietta, GA',
                        'Austin-Round Rock-San Marcos, TX':'Austin-Round Rock, TX',
                        'Bakersfield-Delano, CA':'Bakersfield, CA',
                        'Baltimore-Towson, MD':'Baltimore-Columbia-Towson, MD',
                        'Boise City, ID':'Boise City-Nampa, ID',
                        'Boston-Cambridge-Newton, MA-NH':'Boston-Cambridge-Quincy, MA-NH',
                        'Buffalo-Niagara Falls, NY':'Buffalo-Cheektowaga-Niagara Falls, NY',
                        'Charleston-North Charleston, SC':'Charleston-North Charleston-Summerville, SC',
                        'Charlotte-Gastonia-Concord, NC-SC':'Charlotte-Concord-Gastonia, NC-SC',
                        'Charlotte-Gastonia-Rock Hill, NC-SC':'Charlotte-Concord-Gastonia, NC-SC',
                        'Chicago-Naperville-Elgin, IL-IN-WI':'Chicago-Joliet-Naperville, IL-IN-WI',
                        'Cincinnati, OH-KY-IN':'Cincinnati-Middletown, OH-KY-IN',
                        'Cleveland-Elyria, OH':'Cleveland-Elyria-Mentor, OH',
                        'Denver-Aurora-Broomfield, CO':'Denver-Aurora, CO',
                        'Denver-Aurora-Lakewood, CO':'Denver-Aurora, CO',
                        'Detroit-Warren-Livonia, MI':'Detroit-Warren-Dearborn, MI',
                        'Greenville-Anderson-Mauldin, SC':'Greenville-Mauldin-Easley, SC',
                        'Urban Honolulu, HI':'Honolulu, HI',
                        'Houston-The Woodlands-Sugar Land, TX':'Houston-Sugar Land-Baytown, TX',
                        'Indianapolis-Carmel, IN':'Indianapolis-Carmel-Anderson, IN',
                        'Las Vegas-Paradise, NV':'Las Vegas-Henderson-Paradise, NV',
                        'Los Angeles-Long Beach-Santa Ana, CA':'Los Angeles-Long Beach-Anaheim, CA',
                        'Miami-Fort Lauderdale-Pompano Beach, FL':'Miami-Fort Lauderdale-Miami Beach, FL',
                        'Miami-Fort Lauderdale-West Palm Beach, FL':'Miami-Fort Lauderdale-Miami Beach, FL',
                        'New Orleans-Metairie, LA':'New Orleans-Metairie-Kenner, LA',
                        'New York-Newark-Jersey City, NY-NJ-PA':'New York-Northern New Jersey-Long Island, NY-NJ-PA',
                        'North Port-Bradenton-Sarasota, FL':'North Port-Sarasota-Bradenton, FL',
                        'Orlando-Kissimmee, FL':'Orlando-Kissimmee-Sanford, FL',
                        'Phoenix-Mesa-Glendale, AZ':'Phoenix-Mesa-Scottsdale, AZ',
                        'Portland-South Portland, ME':'Portland-South Portland-Biddeford, ME',
                        'Portland-Vancouver-Hillsboro, OR-WA':'Portland-Vancouver-Beaverton, OR-WA',
                        'Providence-Warwick, RI-MA':'Providence-New Bedford-Fall River, RI-MA',
                        'Raleigh, NC':'Raleigh-Cary, NC',
                        'San Antonio, TX':'San Antonio-New Braunfels, TX',
                        'San Diego-Carlsbad, CA':'San Diego-Carlsbad-San Marcos, CA',
                        'San Francisco-Oakland-Hayward, CA':'San Francisco-Oakland-Fremont, CA',
                        'Stockton, CA':'Stockton-Lodi, CA',
                        'Worcester, MA':'Worcester, MA-CT'}

msa_corr_map_to_abbr = {'Akron, OH':'AKRON_OH',
                        'Albany-Schenectady-Troy, NY':'ALBANY_NY',
                        'Albuquerque, NM':'ALBUQUERQUE_NM',
                        'Allentown-Bethlehem-Easton, PA-NJ':'ALLENTOWN_PA',
                        'Atlanta-Sandy Springs-Marietta, GA':'ATLANTA_GA',
                        'Augusta-Richmond County, GA-SC':'AUGUSTA_GA',
                        'Austin-Round Rock, TX':'AUSTIN_TX',
                        'Bakersfield, CA':'BAKERSFIELD_CA',
                        'Baltimore-Columbia-Towson, MD':'BALTIMORE_MD',
                        'Baton Rouge, LA':'BATON_ROUGE_LA',
                        'Birmingham-Hoover, AL':'BIRMINGHAM_AL',
                        'Boise City-Nampa, ID':'BOISE_ID',
                        'Boston-Cambridge-Quincy, MA-NH':'BOSTON_MA',
                        'Bradenton-Sarasota-Venice, FL':'BRADENTON_FL',
                        'Bridgeport-Stamford-Norwalk, CT':'BRIDGEPORT_CT',
                        'Buffalo-Cheektowaga-Niagara Falls, NY':'BUFFALO_NY',
                        'Cape Coral-Fort Myers, FL':'CAPE_CORAL_FL',
                        'Charleston-North Charleston-Summerville, SC':'CHARLESTON_SC',
                        'Charlotte-Concord-Gastonia, NC-SC':'CHARLOTTE_NC',
                        'Chattanooga, TN-GA':'CHATANOOGA_TN',
                        'Chicago-Joliet-Naperville, IL-IN-WI':'CHICAGO_IL',
                        'Cincinnati-Middletown, OH-KY-IN':'CINCINNATI_OH',
                        'Cleveland-Elyria-Mentor, OH':'CLEVELAND_OH',
                        'Colorado Springs, CO':'COLORADO_SPRINGS_CO',
                        'Columbia, SC':'COLUMBIA_SC',
                        'Columbus, OH':'COLUMBUS_OH',
                        'Dallas-Fort Worth-Arlington, TX':'DALLAS_TX',
                        'Dayton, OH':'DAYTON_OH',
                        'Deltona-Daytona Beach-Ormond Beach, FL':'DELTONA_FL',
                        'Denver-Aurora, CO':'DENVER_CO',
                        'Des Moines-West Des Moines, IA':'DES_MOINES_IA',
                        'Detroit-Warren-Dearborn, MI':'DETROIT_MI',
                        'Durham-Chapel Hill, NC':'DURHAM_NC',
                        'El Paso, TX':'EL_PASO_TX',
                        'Fresno, CA':'FRESNO_CA',
                        'Grand Rapids-Wyoming, MI':'GRAND_RAPIDS_MI',
                        'Greensboro-High Point, NC':'GREENSBORO_NC',
                        'Greenville-Mauldin-Easley, SC':'GREENVILLE_SC',
                        'Harrisburg-Carlisle, PA':'HARRISBURG_PA',
                        'Hartford-West Hartford-East Hartford, CT':'HARTFORD_CT',
                        'Honolulu, HI':'HONOLULU_HI',
                        'Houston-Sugar Land-Baytown, TX':'HOUSTON_TX',
                        'Indianapolis-Carmel-Anderson, IN':'INDIANAPOLIS_IN',
                        'Jackson, MS':'JACKSON_MS',
                        'Jacksonville, FL':'JACKSONVILLE_FL',
                        'Kansas City, MO-KS':'KANSAS_CITY_MO',
                        'Knoxville, TN':'KNOXVILLE_TN',
                        'Lakeland-Winter Haven, FL':'LAKELAND_FL',
                        'Lancaster, PA':'LANCASTER_PA',
                        'Las Vegas-Henderson-Paradise, NV':'LAS_VEGAS_NV',
                        'Lexington-Fayette, KY':'LEXINGTON_KY',
                        'Little Rock-North Little Rock-Conway, AR':'LITTLE_ROCK_AR',
                        'Los Angeles-Long Beach-Anaheim, CA':'LOS_ANGELES_CA',
                        'Louisville-Jefferson County, KY-IN':'LOUISVILLE_KY',
                        'Louisville/Jefferson County, KY-IN':'LOUISVILLE_KY',
                        'Madison, WI':'MADISON_WI',
                        'McAllen-Edinburg-Mission, TX':'MCALLEN_TX',
                        'Memphis, TN-MS-AR':'MEMPHIS_TN',
                        'Miami-Fort Lauderdale-Miami Beach, FL':'MIAMI_FL',
                        'Milwaukee-Waukesha-West Allis, WI':'MILWAUKEE_WI',
                        'Minneapolis-St. Paul-Bloomington, MN-WI':'MINNEAPOLIS_MN',
                        'Modesto, CA':'MODESTO_CA',
                        'Nashville-Davidson--Murfreesboro--Franklin, TN':'NASHVILLE_TN',
                        'New Haven-Milford, CT':'NEW_HAVEN_CT',
                        'New Orleans-Metairie-Kenner, LA':'NEW_ORLEANS_LA',
                        'New York-Northern New Jersey-Long Island, NY-NJ-PA':'NEW_YORK_NY',
                        'North Port-Sarasota-Bradenton, FL':'NORTH_PORT_FL',
                        'Ogden-Clearfield, UT':'OGDEN_UT',
                        'Oklahoma City, OK':'OKLAHOMA_CITY_OK',
                        'Omaha-Council Bluffs, NE-IA':'OMAHA_NE',
                        'Orlando-Kissimmee-Sanford, FL':'ORLANDO_FL',
                        'Oxnard-Thousand Oaks-Ventura, CA':'OXNARD_CA',
                        'Palm Bay-Melbourne-Titusville, FL':'PALM_BAY_FL',
                        'Philadelphia-Camden-Wilmington, PA-NJ-DE-MD':'PHILADELPHIA_PA',
                        'Phoenix-Mesa-Scottsdale, AZ':'PHOENIX_AZ',
                        'Pittsburgh, PA':'PITTSBURGH_PA',
                        'Portland-South Portland-Biddeford, ME':'PORTLAND_ME',
                        'Portland-Vancouver-Beaverton, OR-WA':'PORTLAND_OR',
                        'Poughkeepsie-Newburgh-Middletown, NY':'POUGHKEEPSIE_NY',
                        'Providence-New Bedford-Fall River, RI-MA':'PROVIDENCE_RI',
                        'Provo-Orem, UT':'PROVO_UT',
                        'Raleigh-Cary, NC':'RALEIGH_NC',
                        'Richmond, VA':'RICHMOND_VA',
                        'Riverside-San Bernardino-Ontario, CA':'RIVERSIDE_CA',
                        'Rochester, NY':'ROCHESTER_NY',
                        'Salt Lake City, UT':'SALT_LAKE_CITY_UT',
                        'San Antonio-New Braunfels, TX':'SAN_ANTONIO_TX',
                        'San Diego-Carlsbad-San Marcos, CA':'SAN_DIEGO_CA',
                        'San Francisco-Oakland-Fremont, CA':'SAN_FRANCISCO_CA',
                        'San Jose-Sunnyvale-Santa Clara, CA':'SAN_JOSE_CA',
                        'Santa Rosa, CA':'SANTA_ROSA_CA',
                        'Seattle-Tacoma-Bellevue, WA':'SEATTLE_WA',
                        'Spokane-Spokane Valley, WA':'SPOKANE_WA',
                        'Springfield, MA':'SPRINGFIELD_MA',
                        'St. Louis, MO-IL':'ST_LOUIS_MO',
                        'Stockton-Lodi, CA':'STOCKTON_CA',
                        'Syracuse, NY':'SYRACUSE_NY',
                        'Tampa-St. Petersburg-Clearwater, FL':'TAMPA_FL',
                        'Toledo, OH':'TOLEDO_OH',
                        'Tucson, AZ':'TUCSON_AZ',
                        'Tulsa, OK':'TULSA_OK',
                        'Virginia Beach-Norfolk-Newport News, VA-NC':'VIRGINIA_BEACH_NC',
                        'Washington-Arlington-Alexandria, DC-VA-MD-WV':'WASHINGTON_DC',
                        'Wichita, KS':'WICHITA_KS',
                        'Winston-Salem, NC':'WINSTON_NC',
                        'Worcester, MA-CT':'WORCESTER_MA',
                        'Youngstown-Warren-Boardman, OH-PA':'YOUNGSTOWN_OH'}

```




```python
# process MSAs

#rename msa to msa_orig ~~~~~~~~~~~~~~~~~~~~~~~$$$$$
all_df = all_df.rename(columns={'MSA':'MSA_orig'})

rename_msa = lambda x : msa_orig_map_to_corr.get(x)


all_df['MSA_corr'] = all_df['MSA_orig']\
    .apply(lambda x :msa_orig_map_to_corr.get(x) if msa_orig_map_to_corr.get(x) is not None else x)
all_df['MSA_abbr'] = all_df['MSA_corr'].apply(lambda x : msa_corr_map_to_abbr.get(x))


columns = ['MSA_orig', 'MSA_corr', 'MSA_abbr', 'year',
           'now_married_except_separated',
           'less_than_high_school_diploma',
           'unmarried_portion_of_women_15_to_50_years_who_had_a_birth_in_past_12_months',
           'households_with_food_stamp_snap_benefits',
           'percentage_married-couple_family',
           'percentage_female_householder_no_husband_present_family',
           'poverty_all_people', 'house_median_value_(dollars)',
           'murder_per_100_k']

all_df = all_df[columns]
all_df
```





<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>MSA_orig</th>
      <th>MSA_corr</th>
      <th>MSA_abbr</th>
      <th>year</th>
      <th>now_married_except_separated</th>
      <th>less_than_high_school_diploma</th>
      <th>unmarried_portion_of_women_15_to_50_years_who_had_a_birth_in_past_12_months</th>
      <th>households_with_food_stamp_snap_benefits</th>
      <th>percentage_married-couple_family</th>
      <th>percentage_female_householder_no_husband_present_family</th>
      <th>poverty_all_people</th>
      <th>house_median_value_(dollars)</th>
      <th>murder_per_100_k</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Atlanta-Sandy Springs-Marietta, GA</td>
      <td>Atlanta-Sandy Springs-Marietta, GA</td>
      <td>ATLANTA_GA</td>
      <td>2006</td>
      <td>49.2</td>
      <td>14.2</td>
      <td>34.3</td>
      <td>5.9</td>
      <td>72.6</td>
      <td>20.0</td>
      <td>11.9</td>
      <td>186800</td>
      <td>7.4</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Austin-Round Rock, TX</td>
      <td>Austin-Round Rock, TX</td>
      <td>AUSTIN_TX</td>
      <td>2006</td>
      <td>48.7</td>
      <td>13.7</td>
      <td>30.9</td>
      <td>5.9</td>
      <td>75.0</td>
      <td>17.0</td>
      <td>13.0</td>
      <td>164100</td>
      <td>1.9</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Baltimore-Towson, MD</td>
      <td>Baltimore-Columbia-Towson, MD</td>
      <td>BALTIMORE_MD</td>
      <td>2006</td>
      <td>47.2</td>
      <td>14.0</td>
      <td>31.4</td>
      <td>5.5</td>
      <td>71.8</td>
      <td>21.4</td>
      <td>9.0</td>
      <td>300600</td>
      <td>13.3</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Birmingham-Hoover, AL</td>
      <td>Birmingham-Hoover, AL</td>
      <td>BIRMINGHAM_AL</td>
      <td>2006</td>
      <td>50.9</td>
      <td>15.8</td>
      <td>34.7</td>
      <td>7.1</td>
      <td>72.5</td>
      <td>21.6</td>
      <td>13.1</td>
      <td>131400</td>
      <td>12.5</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Buffalo-Niagara Falls, NY</td>
      <td>Buffalo-Cheektowaga-Niagara Falls, NY</td>
      <td>BUFFALO_NY</td>
      <td>2006</td>
      <td>47.1</td>
      <td>12.9</td>
      <td>38.0</td>
      <td>9.6</td>
      <td>72.8</td>
      <td>20.9</td>
      <td>14.2</td>
      <td>105000</td>
      <td>7.5</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Charlotte-Gastonia-Concord, NC-SC</td>
      <td>Charlotte-Concord-Gastonia, NC-SC</td>
      <td>CHARLOTTE_NC</td>
      <td>2006</td>
      <td>52.1</td>
      <td>14.7</td>
      <td>29.4</td>
      <td>7.3</td>
      <td>72.8</td>
      <td>20.2</td>
      <td>11.5</td>
      <td>157600</td>
      <td>8.3</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Cincinnati-Middletown, OH-KY-IN</td>
      <td>Cincinnati-Middletown, OH-KY-IN</td>
      <td>CINCINNATI_OH</td>
      <td>2006</td>
      <td>50.4</td>
      <td>13.4</td>
      <td>39.8</td>
      <td>7.4</td>
      <td>74.6</td>
      <td>19.4</td>
      <td>11.9</td>
      <td>152100</td>
      <td>5.9</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Cleveland-Elyria-Mentor, OH</td>
      <td>Cleveland-Elyria-Mentor, OH</td>
      <td>CLEVELAND_OH</td>
      <td>2006</td>
      <td>46.8</td>
      <td>13.5</td>
      <td>41.7</td>
      <td>9.0</td>
      <td>70.5</td>
      <td>22.6</td>
      <td>12.5</td>
      <td>149600</td>
      <td>4.7</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Columbus, OH</td>
      <td>Columbus, OH</td>
      <td>COLUMBUS_OH</td>
      <td>2006</td>
      <td>50.7</td>
      <td>11.6</td>
      <td>33.4</td>
      <td>9.2</td>
      <td>74.9</td>
      <td>18.1</td>
      <td>13.1</td>
      <td>162100</td>
      <td>6.6</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Dallas-Fort Worth-Arlington, TX</td>
      <td>Dallas-Fort Worth-Arlington, TX</td>
      <td>DALLAS_TX</td>
      <td>2006</td>
      <td>51.9</td>
      <td>18.8</td>
      <td>27.8</td>
      <td>6.3</td>
      <td>74.4</td>
      <td>18.2</td>
      <td>12.9</td>
      <td>141100</td>
      <td>5.6</td>
    </tr>
    <tr>
      <th>10</th>
      <td>Denver-Aurora, CO</td>
      <td>Denver-Aurora, CO</td>
      <td>DENVER_CO</td>
      <td>2006</td>
      <td>52.6</td>
      <td>12.6</td>
      <td>23.7</td>
      <td>4.1</td>
      <td>76.8</td>
      <td>16.0</td>
      <td>11.5</td>
      <td>245200</td>
      <td>4.4</td>
    </tr>
    <tr>
      <th>11</th>
      <td>Detroit-Warren-Livonia, MI</td>
      <td>Detroit-Warren-Dearborn, MI</td>
      <td>DETROIT_MI</td>
      <td>2006</td>
      <td>48.1</td>
      <td>13.4</td>
      <td>37.8</td>
      <td>9.3</td>
      <td>71.8</td>
      <td>21.3</td>
      <td>12.9</td>
      <td>173400</td>
      <td>11.3</td>
    </tr>
    <tr>
      <th>12</th>
      <td>Hartford-West Hartford-East Hartford, CT</td>
      <td>Hartford-West Hartford-East Hartford, CT</td>
      <td>HARTFORD_CT</td>
      <td>2006</td>
      <td>50.9</td>
      <td>12.3</td>
      <td>26.5</td>
      <td>6.2</td>
      <td>76.8</td>
      <td>17.9</td>
      <td>9.0</td>
      <td>246900</td>
      <td>3.8</td>
    </tr>
    <tr>
      <th>13</th>
      <td>Houston-Sugar Land-Baytown, TX</td>
      <td>Houston-Sugar Land-Baytown, TX</td>
      <td>HOUSTON_TX</td>
      <td>2006</td>
      <td>50.6</td>
      <td>21.2</td>
      <td>33.6</td>
      <td>7.6</td>
      <td>72.7</td>
      <td>19.6</td>
      <td>14.9</td>
      <td>129800</td>
      <td>9.6</td>
    </tr>
    <tr>
      <th>14</th>
      <td>Indianapolis-Carmel, IN</td>
      <td>Indianapolis-Carmel-Anderson, IN</td>
      <td>INDIANAPOLIS_IN</td>
      <td>2006</td>
      <td>51.5</td>
      <td>12.8</td>
      <td>35.9</td>
      <td>7.9</td>
      <td>74.1</td>
      <td>18.9</td>
      <td>11.2</td>
      <td>140300</td>
      <td>9.4</td>
    </tr>
    <tr>
      <th>15</th>
      <td>Jacksonville, FL</td>
      <td>Jacksonville, FL</td>
      <td>JACKSONVILLE_FL</td>
      <td>2006</td>
      <td>50.0</td>
      <td>11.9</td>
      <td>36.2</td>
      <td>5.4</td>
      <td>73.5</td>
      <td>20.2</td>
      <td>11.8</td>
      <td>192800</td>
      <td>10.6</td>
    </tr>
    <tr>
      <th>16</th>
      <td>Kansas City, MO-KS</td>
      <td>Kansas City, MO-KS</td>
      <td>KANSAS_CITY_MO</td>
      <td>2006</td>
      <td>52.8</td>
      <td>10.4</td>
      <td>34.3</td>
      <td>7.5</td>
      <td>75.7</td>
      <td>18.2</td>
      <td>10.7</td>
      <td>153000</td>
      <td>8.9</td>
    </tr>
    <tr>
      <th>17</th>
      <td>Las Vegas-Paradise, NV</td>
      <td>Las Vegas-Henderson-Paradise, NV</td>
      <td>LAS_VEGAS_NV</td>
      <td>2006</td>
      <td>50.3</td>
      <td>17.1</td>
      <td>29.1</td>
      <td>3.8</td>
      <td>71.5</td>
      <td>18.3</td>
      <td>10.3</td>
      <td>320800</td>
      <td>10.2</td>
    </tr>
    <tr>
      <th>18</th>
      <td>Los Angeles-Long Beach-Santa Ana, CA</td>
      <td>Los Angeles-Long Beach-Anaheim, CA</td>
      <td>LOS_ANGELES_CA</td>
      <td>2006</td>
      <td>46.1</td>
      <td>23.2</td>
      <td>31.3</td>
      <td>4.1</td>
      <td>70.0</td>
      <td>20.8</td>
      <td>14.1</td>
      <td>604500</td>
      <td>8.4</td>
    </tr>
    <tr>
      <th>19</th>
      <td>Louisville-Jefferson County, KY-IN</td>
      <td>Louisville-Jefferson County, KY-IN</td>
      <td>LOUISVILLE_KY</td>
      <td>2006</td>
      <td>49.8</td>
      <td>15.4</td>
      <td>35.2</td>
      <td>9.5</td>
      <td>72.3</td>
      <td>21.1</td>
      <td>13.3</td>
      <td>139000</td>
      <td>4.7</td>
    </tr>
    <tr>
      <th>20</th>
      <td>Memphis, TN-MS-AR</td>
      <td>Memphis, TN-MS-AR</td>
      <td>MEMPHIS_TN</td>
      <td>2006</td>
      <td>43.2</td>
      <td>17.0</td>
      <td>51.2</td>
      <td>13.4</td>
      <td>62.1</td>
      <td>28.9</td>
      <td>17.8</td>
      <td>125600</td>
      <td>13.7</td>
    </tr>
    <tr>
      <th>21</th>
      <td>Miami-Fort Lauderdale-Miami Beach, FL</td>
      <td>Miami-Fort Lauderdale-Miami Beach, FL</td>
      <td>MIAMI_FL</td>
      <td>2006</td>
      <td>47.3</td>
      <td>17.9</td>
      <td>37.7</td>
      <td>12.1</td>
      <td>70.4</td>
      <td>21.9</td>
      <td>13.3</td>
      <td>312500</td>
      <td>7.6</td>
    </tr>
    <tr>
      <th>22</th>
      <td>Milwaukee-Waukesha-West Allis, WI</td>
      <td>Milwaukee-Waukesha-West Allis, WI</td>
      <td>MILWAUKEE_WI</td>
      <td>2006</td>
      <td>48.5</td>
      <td>12.6</td>
      <td>40.8</td>
      <td>7.6</td>
      <td>72.2</td>
      <td>20.6</td>
      <td>13.0</td>
      <td>197300</td>
      <td>7.4</td>
    </tr>
    <tr>
      <th>23</th>
      <td>Minneapolis-St. Paul-Bloomington, MN-WI</td>
      <td>Minneapolis-St. Paul-Bloomington, MN-WI</td>
      <td>MINNEAPOLIS_MN</td>
      <td>2006</td>
      <td>52.4</td>
      <td>7.8</td>
      <td>24.6</td>
      <td>4.7</td>
      <td>78.6</td>
      <td>15.1</td>
      <td>8.9</td>
      <td>242100</td>
      <td>3.5</td>
    </tr>
    <tr>
      <th>24</th>
      <td>New Orleans-Metairie-Kenner, LA</td>
      <td>New Orleans-Metairie-Kenner, LA</td>
      <td>NEW_ORLEANS_LA</td>
      <td>2006</td>
      <td>47.0</td>
      <td>17.4</td>
      <td>41.3</td>
      <td>30.6</td>
      <td>71.8</td>
      <td>20.3</td>
      <td>14.8</td>
      <td>170200</td>
      <td>21.7</td>
    </tr>
    <tr>
      <th>25</th>
      <td>New York-Northern New Jersey-Long Island, NY-N...</td>
      <td>New York-Northern New Jersey-Long Island, NY-N...</td>
      <td>NEW_YORK_NY</td>
      <td>2006</td>
      <td>47.2</td>
      <td>16.4</td>
      <td>27.4</td>
      <td>8.1</td>
      <td>70.0</td>
      <td>22.6</td>
      <td>12.8</td>
      <td>458700</td>
      <td>5.2</td>
    </tr>
    <tr>
      <th>26</th>
      <td>Oklahoma City, OK</td>
      <td>Oklahoma City, OK</td>
      <td>OKLAHOMA_CITY_OK</td>
      <td>2006</td>
      <td>50.9</td>
      <td>14.6</td>
      <td>33.6</td>
      <td>9.8</td>
      <td>75.7</td>
      <td>17.1</td>
      <td>15.1</td>
      <td>109600</td>
      <td>7.0</td>
    </tr>
    <tr>
      <th>27</th>
      <td>Orlando-Kissimmee, FL</td>
      <td>Orlando-Kissimmee-Sanford, FL</td>
      <td>ORLANDO_FL</td>
      <td>2006</td>
      <td>51.5</td>
      <td>13.3</td>
      <td>38.8</td>
      <td>5.3</td>
      <td>73.5</td>
      <td>19.2</td>
      <td>11.1</td>
      <td>243100</td>
      <td>8.1</td>
    </tr>
    <tr>
      <th>28</th>
      <td>Philadelphia-Camden-Wilmington, PA-NJ-DE-MD</td>
      <td>Philadelphia-Camden-Wilmington, PA-NJ-DE-MD</td>
      <td>PHILADELPHIA_PA</td>
      <td>2006</td>
      <td>46.6</td>
      <td>13.5</td>
      <td>38.5</td>
      <td>6.6</td>
      <td>71.7</td>
      <td>21.3</td>
      <td>11.8</td>
      <td>230300</td>
      <td>9.5</td>
    </tr>
    <tr>
      <th>29</th>
      <td>Phoenix-Mesa-Scottsdale, AZ</td>
      <td>Phoenix-Mesa-Scottsdale, AZ</td>
      <td>PHOENIX_AZ</td>
      <td>2006</td>
      <td>50.3</td>
      <td>16.1</td>
      <td>31.8</td>
      <td>5.5</td>
      <td>74.9</td>
      <td>16.8</td>
      <td>12.7</td>
      <td>266300</td>
      <td>8.7</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>898</th>
      <td>Philadelphia-Camden-Wilmington, PA-NJ-DE-MD</td>
      <td>Philadelphia-Camden-Wilmington, PA-NJ-DE-MD</td>
      <td>PHILADELPHIA_PA</td>
      <td>2016</td>
      <td>44.7</td>
      <td>9.8</td>
      <td>39.9</td>
      <td>12.6</td>
      <td>70.7</td>
      <td>21.7</td>
      <td>12.9</td>
      <td>245600</td>
      <td>7.9</td>
    </tr>
    <tr>
      <th>899</th>
      <td>Phoenix-Mesa-Scottsdale, AZ</td>
      <td>Phoenix-Mesa-Scottsdale, AZ</td>
      <td>PHOENIX_AZ</td>
      <td>2016</td>
      <td>46.9</td>
      <td>12.9</td>
      <td>33.7</td>
      <td>11.0</td>
      <td>72.4</td>
      <td>18.7</td>
      <td>15.0</td>
      <td>231000</td>
      <td>5.5</td>
    </tr>
    <tr>
      <th>900</th>
      <td>Pittsburgh, PA</td>
      <td>Pittsburgh, PA</td>
      <td>PITTSBURGH_PA</td>
      <td>2016</td>
      <td>48.3</td>
      <td>6.5</td>
      <td>35.3</td>
      <td>12.4</td>
      <td>76.5</td>
      <td>17.1</td>
      <td>10.8</td>
      <td>148600</td>
      <td>5.1</td>
    </tr>
    <tr>
      <th>901</th>
      <td>Portland-South Portland, ME</td>
      <td>Portland-South Portland-Biddeford, ME</td>
      <td>PORTLAND_ME</td>
      <td>2016</td>
      <td>49.7</td>
      <td>5.7</td>
      <td>21.4</td>
      <td>10.1</td>
      <td>79.3</td>
      <td>13.8</td>
      <td>9.1</td>
      <td>251300</td>
      <td>1.5</td>
    </tr>
    <tr>
      <th>902</th>
      <td>Portland-Vancouver-Hillsboro, OR-WA</td>
      <td>Portland-Vancouver-Beaverton, OR-WA</td>
      <td>PORTLAND_OR</td>
      <td>2016</td>
      <td>49.9</td>
      <td>8.2</td>
      <td>25.8</td>
      <td>13.8</td>
      <td>78.3</td>
      <td>15.0</td>
      <td>10.9</td>
      <td>345000</td>
      <td>1.7</td>
    </tr>
    <tr>
      <th>903</th>
      <td>Providence-Warwick, RI-MA</td>
      <td>Providence-New Bedford-Fall River, RI-MA</td>
      <td>PROVIDENCE_RI</td>
      <td>2016</td>
      <td>44.0</td>
      <td>12.4</td>
      <td>48.2</td>
      <td>15.8</td>
      <td>70.5</td>
      <td>22.0</td>
      <td>12.0</td>
      <td>265300</td>
      <td>2.4</td>
    </tr>
    <tr>
      <th>904</th>
      <td>Provo-Orem, UT</td>
      <td>Provo-Orem, UT</td>
      <td>PROVO_UT</td>
      <td>2016</td>
      <td>57.2</td>
      <td>5.5</td>
      <td>14.9</td>
      <td>7.1</td>
      <td>86.4</td>
      <td>9.3</td>
      <td>11.6</td>
      <td>277000</td>
      <td>0.8</td>
    </tr>
    <tr>
      <th>905</th>
      <td>Riverside-San Bernardino-Ontario, CA</td>
      <td>Riverside-San Bernardino-Ontario, CA</td>
      <td>RIVERSIDE_CA</td>
      <td>2016</td>
      <td>46.2</td>
      <td>20.0</td>
      <td>40.2</td>
      <td>13.6</td>
      <td>70.4</td>
      <td>20.4</td>
      <td>16.4</td>
      <td>318900</td>
      <td>5.1</td>
    </tr>
    <tr>
      <th>906</th>
      <td>Rochester, NY</td>
      <td>Rochester, NY</td>
      <td>ROCHESTER_NY</td>
      <td>2016</td>
      <td>44.9</td>
      <td>9.9</td>
      <td>46.4</td>
      <td>14.2</td>
      <td>71.7</td>
      <td>21.7</td>
      <td>13.9</td>
      <td>138900</td>
      <td>4.7</td>
    </tr>
    <tr>
      <th>907</th>
      <td>St. Louis, MO-IL</td>
      <td>St. Louis, MO-IL</td>
      <td>ST_LOUIS_MO</td>
      <td>2016</td>
      <td>48.1</td>
      <td>8.2</td>
      <td>39.2</td>
      <td>10.5</td>
      <td>72.6</td>
      <td>20.5</td>
      <td>11.4</td>
      <td>169200</td>
      <td>11.1</td>
    </tr>
    <tr>
      <th>908</th>
      <td>Salt Lake City, UT</td>
      <td>Salt Lake City, UT</td>
      <td>SALT_LAKE_CITY_UT</td>
      <td>2016</td>
      <td>51.1</td>
      <td>10.0</td>
      <td>16.1</td>
      <td>6.9</td>
      <td>76.4</td>
      <td>16.0</td>
      <td>9.1</td>
      <td>267800</td>
      <td>3.9</td>
    </tr>
    <tr>
      <th>909</th>
      <td>San Antonio-New Braunfels, TX</td>
      <td>San Antonio-New Braunfels, TX</td>
      <td>SAN_ANTONIO_TX</td>
      <td>2016</td>
      <td>46.1</td>
      <td>16.2</td>
      <td>37.0</td>
      <td>12.1</td>
      <td>70.8</td>
      <td>22.3</td>
      <td>15.0</td>
      <td>160200</td>
      <td>7.7</td>
    </tr>
    <tr>
      <th>910</th>
      <td>San Diego-Carlsbad, CA</td>
      <td>San Diego-Carlsbad-San Marcos, CA</td>
      <td>SAN_DIEGO_CA</td>
      <td>2016</td>
      <td>47.3</td>
      <td>13.6</td>
      <td>25.3</td>
      <td>7.5</td>
      <td>74.7</td>
      <td>18.0</td>
      <td>12.3</td>
      <td>527600</td>
      <td>3.0</td>
    </tr>
    <tr>
      <th>911</th>
      <td>San Francisco-Oakland-Hayward, CA</td>
      <td>San Francisco-Oakland-Fremont, CA</td>
      <td>SAN_FRANCISCO_CA</td>
      <td>2016</td>
      <td>48.8</td>
      <td>11.3</td>
      <td>22.4</td>
      <td>5.5</td>
      <td>76.5</td>
      <td>16.2</td>
      <td>9.2</td>
      <td>796100</td>
      <td>5.1</td>
    </tr>
    <tr>
      <th>912</th>
      <td>San Jose-Sunnyvale-Santa Clara, CA</td>
      <td>San Jose-Sunnyvale-Santa Clara, CA</td>
      <td>SAN_JOSE_CA</td>
      <td>2016</td>
      <td>53.5</td>
      <td>13.0</td>
      <td>14.4</td>
      <td>5.5</td>
      <td>79.1</td>
      <td>14.0</td>
      <td>9.4</td>
      <td>911900</td>
      <td>2.9</td>
    </tr>
    <tr>
      <th>913</th>
      <td>Santa Rosa, CA</td>
      <td>Santa Rosa, CA</td>
      <td>SANTA_ROSA_CA</td>
      <td>2016</td>
      <td>46.6</td>
      <td>12.1</td>
      <td>32.8</td>
      <td>6.8</td>
      <td>75.2</td>
      <td>15.6</td>
      <td>9.2</td>
      <td>565200</td>
      <td>2.0</td>
    </tr>
    <tr>
      <th>914</th>
      <td>Seattle-Tacoma-Bellevue, WA</td>
      <td>Seattle-Tacoma-Bellevue, WA</td>
      <td>SEATTLE_WA</td>
      <td>2016</td>
      <td>50.1</td>
      <td>8.0</td>
      <td>20.9</td>
      <td>10.5</td>
      <td>77.7</td>
      <td>15.2</td>
      <td>9.6</td>
      <td>391500</td>
      <td>2.7</td>
    </tr>
    <tr>
      <th>915</th>
      <td>Spokane-Spokane Valley, WA</td>
      <td>Spokane-Spokane Valley, WA</td>
      <td>SPOKANE_WA</td>
      <td>2016</td>
      <td>49.1</td>
      <td>6.3</td>
      <td>23.1</td>
      <td>15.9</td>
      <td>76.1</td>
      <td>16.7</td>
      <td>13.0</td>
      <td>200200</td>
      <td>2.9</td>
    </tr>
    <tr>
      <th>916</th>
      <td>Springfield, MA</td>
      <td>Springfield, MA</td>
      <td>SPRINGFIELD_MA</td>
      <td>2016</td>
      <td>39.1</td>
      <td>12.3</td>
      <td>43.6</td>
      <td>17.1</td>
      <td>64.7</td>
      <td>27.0</td>
      <td>15.6</td>
      <td>219800</td>
      <td>2.9</td>
    </tr>
    <tr>
      <th>917</th>
      <td>Stockton-Lodi, CA</td>
      <td>Stockton-Lodi, CA</td>
      <td>STOCKTON_CA</td>
      <td>2016</td>
      <td>46.2</td>
      <td>22.9</td>
      <td>45.3</td>
      <td>15.8</td>
      <td>67.4</td>
      <td>21.4</td>
      <td>14.4</td>
      <td>316000</td>
      <td>8.5</td>
    </tr>
    <tr>
      <th>918</th>
      <td>Syracuse, NY</td>
      <td>Syracuse, NY</td>
      <td>SYRACUSE_NY</td>
      <td>2016</td>
      <td>44.7</td>
      <td>10.0</td>
      <td>45.2</td>
      <td>14.6</td>
      <td>73.6</td>
      <td>18.4</td>
      <td>14.7</td>
      <td>133300</td>
      <td>5.3</td>
    </tr>
    <tr>
      <th>919</th>
      <td>Tampa-St. Petersburg-Clearwater, FL</td>
      <td>Tampa-St. Petersburg-Clearwater, FL</td>
      <td>TAMPA_FL</td>
      <td>2016</td>
      <td>45.9</td>
      <td>11.1</td>
      <td>39.7</td>
      <td>13.6</td>
      <td>72.2</td>
      <td>19.9</td>
      <td>14.2</td>
      <td>175200</td>
      <td>3.9</td>
    </tr>
    <tr>
      <th>920</th>
      <td>Toledo, OH</td>
      <td>Toledo, OH</td>
      <td>TOLEDO_OH</td>
      <td>2016</td>
      <td>42.3</td>
      <td>9.5</td>
      <td>50.2</td>
      <td>16.3</td>
      <td>67.2</td>
      <td>22.3</td>
      <td>17.5</td>
      <td>129200</td>
      <td>7.1</td>
    </tr>
    <tr>
      <th>921</th>
      <td>Tucson, AZ</td>
      <td>Tucson, AZ</td>
      <td>TUCSON_AZ</td>
      <td>2016</td>
      <td>44.8</td>
      <td>12.0</td>
      <td>53.3</td>
      <td>13.7</td>
      <td>70.6</td>
      <td>22.1</td>
      <td>18.4</td>
      <td>170300</td>
      <td>4.8</td>
    </tr>
    <tr>
      <th>922</th>
      <td>Tulsa, OK</td>
      <td>Tulsa, OK</td>
      <td>TULSA_OK</td>
      <td>2016</td>
      <td>49.4</td>
      <td>10.6</td>
      <td>35.3</td>
      <td>12.4</td>
      <td>72.6</td>
      <td>19.5</td>
      <td>15.1</td>
      <td>143400</td>
      <td>9.2</td>
    </tr>
    <tr>
      <th>923</th>
      <td>Urban Honolulu, HI</td>
      <td>Honolulu, HI</td>
      <td>HONOLULU_HI</td>
      <td>2016</td>
      <td>49.8</td>
      <td>8.4</td>
      <td>28.4</td>
      <td>9.8</td>
      <td>76.2</td>
      <td>16.3</td>
      <td>8.5</td>
      <td>658900</td>
      <td>1.6</td>
    </tr>
    <tr>
      <th>924</th>
      <td>Virginia Beach-Norfolk-Newport News, VA-NC</td>
      <td>Virginia Beach-Norfolk-Newport News, VA-NC</td>
      <td>VIRGINIA_BEACH_NC</td>
      <td>2016</td>
      <td>45.4</td>
      <td>8.9</td>
      <td>30.8</td>
      <td>9.7</td>
      <td>70.0</td>
      <td>23.2</td>
      <td>11.4</td>
      <td>239900</td>
      <td>9.5</td>
    </tr>
    <tr>
      <th>925</th>
      <td>Washington-Arlington-Alexandria, DC-VA-MD-WV</td>
      <td>Washington-Arlington-Alexandria, DC-VA-MD-WV</td>
      <td>WASHINGTON_DC</td>
      <td>2016</td>
      <td>48.6</td>
      <td>9.4</td>
      <td>24.9</td>
      <td>7.3</td>
      <td>74.9</td>
      <td>18.0</td>
      <td>8.4</td>
      <td>411400</td>
      <td>4.5</td>
    </tr>
    <tr>
      <th>926</th>
      <td>Wichita, KS</td>
      <td>Wichita, KS</td>
      <td>WICHITA_KS</td>
      <td>2016</td>
      <td>50.6</td>
      <td>10.4</td>
      <td>35.7</td>
      <td>9.7</td>
      <td>76.5</td>
      <td>16.8</td>
      <td>14.0</td>
      <td>132400</td>
      <td>6.5</td>
    </tr>
    <tr>
      <th>927</th>
      <td>Worcester, MA-CT</td>
      <td>Worcester, MA-CT</td>
      <td>WORCESTER_MA</td>
      <td>2016</td>
      <td>45.7</td>
      <td>10.3</td>
      <td>32.2</td>
      <td>12.6</td>
      <td>73.4</td>
      <td>19.4</td>
      <td>9.9</td>
      <td>258600</td>
      <td>1.4</td>
    </tr>
  </tbody>
</table>
<p>928 rows × 13 columns</p>
</div>





```python
all_df.columns
```





    Index(['MSA_orig', 'MSA_corr', 'MSA_abbr', 'year',
           'now_married_except_separated', 'less_than_high_school_diploma',
           'unmarried_portion_of_women_15_to_50_years_who_had_a_birth_in_past_12_months',
           'households_with_food_stamp_snap_benefits',
           'percentage_married-couple_family',
           'percentage_female_householder_no_husband_present_family',
           'poverty_all_people', 'house_median_value_(dollars)',
           'murder_per_100_k'],
          dtype='object')





```python
all_df.to_csv("../data/merged/all_data_2006_to_2016.csv", sep=',',index=False)
all_df.to_pickle("../data/merged/all_data_2006_to_2016.pkl")
```




```python

```

