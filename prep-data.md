Prepare data
================
2021-07-16

``` r
library(tidyverse) # general data manipulation and graphing
library(geoclient) # geocoding tool
library(DBI) # connecting R to database management systems

con <- furmanr::fcdb_connect()

geoclient_api_keys(
  Sys.getenv("GEOCLIENT_APP_ID"),
  Sys.getenv("GEOCLIENT_APP_KEY")
)
```

### Data Sources: Hotel Properties

We retrieve data from the [New York City Department of
Finance](https://www1.nyc.gov/site/finance/index.page), which assesses
each property in New York City for the purpose of, among other things,
calculating property taxes. Using Property Tax System data, this we
focus on properties with hotel building classifications as defined
below. To account for hotels located on lots that also contain condos,
we use a crosswalk file to obtain all BBLs associated with condo lots.
We use the non-condo BBls to join in geographic information from
MapPLUTO.

-   The hotel classes outside of H1-H8, including boutique hotels,
    hostels, extended stay hotels, and SROs (HR) were combined into the
    category we’ve titled “Miscellaneous”. We exclude H5, H6, H7, and
    H8 - private club, apartment hotels, apartment hotel-coops, and
    dormitories - from our analysis.

-   We retrieved union status information from the [Hotel Trades
    Council](https://hotelworkers.org/#find-union-hotels). Addresses
    were run through [the City of New York’s
    geocoder](https://developer.cityofnewyork.us/api/geoclient-api) to
    obtain a mapable BBL. This union status is indicated in the data by
    `is_union`.

-   Under Article 1 Chapter 5, certain hotels are eligible for
    conversion to residential use notwithstanding otherwise relevant
    regulations. This potential eligibility is indicated in the data by
    `article1_eligible`.

``` r
# reading in raw rpad data 
rpad_raw <- tbl(con, "rpad_raw_with_pts") %>%
  filter(
    str_detect(bldgcl,'^H.*') |  bldgcl == 'RH',
    year4 == '2020'
  ) 

xwalk <- tbl(con, "rpad_condo_xwalk") %>%
  filter(year =='2018') %>%
  transmute(
    bbl = bbl_unit,
    bbl_condo
  ) 

raw <- rpad_raw %>%
  filter(
    year4 == '2020'
  ) %>%
  select(
    bldgcl,
    bbl
  ) %>%
  left_join(xwalk, by = "bbl")%>%
  collect()
```

``` r
# reading in raw union data, geocoding
unionhotel_raw <- read_csv("data/unionhotels.csv") %>%
  filter(state == "NY") %>%
  unite(hotel_addr, address, zip, sep = " ,",remove = FALSE)

unionhotel_geocoded <- unionhotel_raw %>%
  geo_search_data(hotel_addr) %>%
  filter(!no_results)%>%
  select(bbl, input_location)%>%
  rename(hotel_addr= input_location) %>%
  right_join(unionhotel_raw, by="hotel_addr")%>%
  select(bbl)%>%
  mutate(is_union = TRUE)%>%
  filter(!is.na(bbl))

article1_cds <-
  c(101, 102, 103, 104, 105, 106, 301, 302, 306, 307, 401, 402) # eligible community districts under art 1 chap 5
```

### Data Sources: Hotel Room Counts

The room count in our data, `final_rooms`, comes from two main sources:
the Department of Finance’s Notice of Property Value records and
Hotels.com. The Notice of Property Value (NOPV) informs every property
owner of the Department of Finance’s assessment of property for the
coming tax year. Also listed in these publicly available PDFs are hotel
room counts for most hotel classified properties. These files are
available through the [Department of Finance’s Property Tax Public
Access web
portal](https://a836-pts-access.nyc.gov/care/forms/htmlframe.aspx?mode=content/home.htm).

-   Some January 2021 NOPVs did not contain room counts, though the
    previous year’s NOPV did. In these cases we resorted to January 2020
    NOPV room counts if available.

-   For lots containing condos, we took the sum total number of hotel
    rooms located on that lot. This allows us to capture the total
    number of hotel rooms on each mappable lot, though there may
    actually be more than one distinct hotel property on that lot. All
    condo BBLs are listed in `condo_bbls`.

``` r
# reading in raw nopv data, combining
nopv21 <- read_csv("J:/DEPT/REUP/Projects/Hotels/nopv/dof-nopv-hotels-20210115.csv") %>%
  transmute(bbl = as.character(bbl),
            hotel_class,
            nopv21_rooms= ifelse(hotel_rooms < 1, NA_real_, hotel_rooms))

nopv20 <- read_csv("J:/DEPT/REUP/Projects/Hotels/nopv/dof-nopv-hotels-20200115.csv") %>%
  transmute(bbl = as.character(bbl),
            nopv20_rooms= ifelse(hotel_rooms < 1, NA_real_, hotel_rooms))

nopv_combined <- nopv20 %>%
  right_join(nopv21, by= "bbl")
```

``` r
# assigning nopv room counts to lots
rpad_hotels <-  raw %>%
  left_join(nopv_combined, by = "bbl") %>% 
  mutate(nopv_rooms = coalesce(nopv21_rooms, nopv20_rooms))%>%
  select(-nopv21_rooms, -nopv20_rooms) %>%
  group_by(bbl_condo) %>%  
  mutate(
    map_nopv_rooms = case_when(
      is.na(bbl_condo) ~ nopv_rooms, # no condos on main lot
      TRUE ~ sum(unique(nopv_rooms), na.rm = TRUE) # condos on main lot 
    ),
    map_bbl = ifelse(is.na(bbl_condo), bbl, bbl_condo)
  ) %>% 
  ungroup() %>%
  distinct(map_bbl,nopv_rooms, .keep_all = TRUE) %>%
  group_by(map_bbl) %>% 
  mutate(
    condo_bbls = paste(bbl, collapse = ";"),
    num_condo_bbls = n_distinct(bbl)
  ) %>% 
  ungroup() %>% 
  distinct(map_bbl, .keep_all = TRUE) %>%
  select(-bbl, -bbl_condo, -nopv_rooms) %>% 
  rename(bbl = map_bbl, nopv_rooms = map_nopv_rooms)

dbWriteTable(con, "rpad_hotels", rpad_hotels, temporary=TRUE, overwrite = TRUE)

# combine MapPLUTO geographic data
rpad_with_pluto <- dbGetQuery(con, "
                              SELECT bldgclass, address, borough, cd, zonedist1, yearbuilt, builtfar, latitude, longitude, r.*
                              FROM mappluto_21v1 p
                              RIGHT JOIN rpad_hotels r
                                USING(bbl)
                              ")
```

While NOPV room counts are available for most hotel classified
properties, an additional step of filling in missing room counts was
made by scraping information from Hotels.com. Available in the dataset
from this scrape are: `hotel_id` `hotel_rooms` `hotel_close_start` and
`hotel_close_end`. Scraped hotel addresses were also run through [the
City of New York’s
geocoder](https://developer.cityofnewyork.us/api/geoclient-api) to
obtain a mappable BBL. Note that the method of geocoding addresses
pulled from Hotels.com is imperfect, and there is a rough approximation
of rooms at these addresses for which NOPV counts are missing. New
hotels or hotels under construction in particular may not have accurate
counts until completion. Ultimately, some properties still needed to be
searched manually to obtain room counts, and in these cases source
information is included in the `source` column.

The final room count is determined first by the availability of January
2021 NOPV counts, and substituted from January 2020 NOPV counts and
Hotels.com data when needed.

``` r
# reading in manual searches
missing_info <- read_csv("data/manual-searches.csv") %>%
  transmute(bbl = as.character(bbl),
            include,
            roomcount,
            source)
```

``` r
# combining frames, hardcoding issues, filtering classes of interest
final_hotels <-
  dbGetQuery(
    con,
    "SELECT bbl, hotel_id, hotel_rooms, hotel_close_start, hotel_close_end FROM hotels_scraped_clean "
  ) %>%
  right_join(rpad_with_pluto, by = "bbl") %>%
  left_join(missing_info, by = "bbl") %>%
  left_join(unionhotel_geocoded, by = "bbl") %>% 
  mutate(
    yearbuilt = case_when(
      hotel_id == "ho1227757728" ~ 2018, # hardcoding hotel missing year built
      yearbuilt == 0 ~ NA_real_,
      TRUE ~ as.numeric(yearbuilt)
    ),
    include = ifelse(is.na(include), TRUE, include),
    is_union = ifelse(is.na(is_union), FALSE, is_union),
    nopv_rooms = ifelse(nopv_rooms == 0, NA_real_, nopv_rooms),
    final_rooms = coalesce(nopv_rooms, hotel_rooms, roomcount),
    final_rooms = case_when(
      hotel_id == "ho370651" ~ 176, # hardcoding a hotel that was incorrectly geocoded
      bbl == "1015577502" ~ 109, # hardcoding a hotel that was incorrectly geocoded
      TRUE ~ final_rooms
    )
  ) %>%
  mutate(
    bldgclass_names = case_when(
      bldgcl == "H1" ~ "Luxury Hotel",
      bldgcl == "H2" ~ "Full Service Hotel",
      bldgcl == "H3" ~ "Limited Service; Many Affiliated w/ National Chain",
      bldgcl == "H4" ~ "Motel",
      bldgcl == "H5" ~ "Hotel; Private Club, Luxury Type",
      bldgcl == "H6" ~ "Apartment Hotel",
      bldgcl == "H7" ~ "Apartment Hotel - Coop",
      bldgcl == "H8" ~ "Dormitory",
      TRUE ~ "Miscellaneous Hotel"
    ),
    main_zone = case_when(
      str_sub(zonedist1, 1, 1) == "C" ~ "Commercial",
      str_sub(zonedist1, 1, 1) == "R" ~ "Residential",
      str_sub(zonedist1, 1, 1) == "M" ~ "Manufacturing",
      zonedist1 == "BPC" ~ "Commercial"
      # Battery Park City is a special district with its own rules, but in looking at the locations
      # of the two hotels there, the subdistricts are close-enough to commercial for these purposes.
    ),
    article1_eligible = case_when(
    (main_zone == "Residential" | main_zone == "Commercial") &
      cd %in% article1_cds &
      yearbuilt < 1961 ~ TRUE,
    TRUE ~ FALSE
  )
  ) %>%
  filter(
    !bldgclass_names %in% c(
      "Apartment Hotel",
      "Apartment Hotel - Coop",
      "Hotel; Private Club, Luxury Type",
      "Dormitory"
    ),
    include != FALSE,!is.na(latitude)
  ) 

write_rds(final_hotels, "data/final_hotels.rds")
```
