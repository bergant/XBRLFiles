# Exploring XBRL files with R
Darko Bergant  
Saturday, January 17, 2015  

#What is XBRL?
Extensible Business Reporting Language ([XBRL](https://www.xbrl.org/the-standard/what/)) is the open international standard for [digital business reporting](http://xbrl.squarespace.com), managed by a global not for profit consortium, [XBRL International](http://xbrl.org). 


#XBRL parser for R
Parsing XBRL is [not](https://www.xbrl.org/the-standard/how/getting-started-for-developers/) something you could do with your eyes closed.
Fortunately the [XBRL](http://cran.r-project.org/web/packages/XBRL) package by Roberto Bertolusso and Marek Kimel takes all the pain away.

To parse complete XBRL, use `xbrlDoAll` function. 
It extracts xbrl instance and related schema files to a list of data frames.


```r
library(XBRL)

inst <- "http://edgar.sec.gov/Archives/edgar/data/21344/000002134414000008/ko-20131231.xml"
options(stringsAsFactors = FALSE)
xbrl.vars <- xbrlDoAll(inst, cache.dir = "XBRLcache", prefix.out = NULL)

str(xbrl.vars, max.level = 1)
```

```
## List of 10
##  $ element     :'data.frame':	21453 obs. of  8 variables:
##  $ role        :'data.frame':	96 obs. of  5 variables:
##  $ calculation :'data.frame':	196 obs. of  11 variables:
##  $ context     :'data.frame':	740 obs. of  13 variables:
##  $ unit        :'data.frame':	4 obs. of  4 variables:
##  $ fact        :'data.frame':	2745 obs. of  7 variables:
##  $ footnote    :'data.frame':	6 obs. of  5 variables:
##  $ definition  :'data.frame':	1398 obs. of  11 variables:
##  $ label       :'data.frame':	2798 obs. of  5 variables:
##  $ presentation:'data.frame':	1582 obs. of  11 variables:
```

# Reading XBRL data frames
The data structure of the data frames is shown in the image below 

![XBRL tables](img/XBRLdiagram.png)

A simple approach to explore the data in interrelated tables is by using [dplyr](http://cran.r-project.org/web/packages/dplyr) package. 
For example, to extract revenue from the sale of goods we have to join *facts* (the numbers) with the 
*contexts* (periods, dimensions):



```r
library(dplyr)

xbrl_sales <-
  xbrl.vars$element %>%
  filter(elementId == "us-gaap_SalesRevenueGoodsNet") %>%
  left_join(xbrl.vars$fact, by = "elementId" ) %>%
  left_join(xbrl.vars$context, by = "contextId") %>%
  filter(is.na(dimension1)) %>%
  select(startDate, endDate, amount = fact, currency = unitId, balance)

knitr::kable(xbrl_sales, format = "markdown")
```



|startDate  |endDate    |amount      |currency |balance |
|:----------|:----------|:-----------|:--------|:-------|
|2011-01-01 |2011-12-31 |46542000000 |usd      |credit  |
|2012-01-01 |2012-12-31 |48017000000 |usd      |credit  |
|2013-01-01 |2013-12-31 |46854000000 |usd      |credit  |

# Presentation: balance sheets example
## Find the statement
XBRL encapsulates several reports. To get the summary 


```r
library(dplyr)

xbrl.vars$role %>%
  group_by(type) %>%
  summarize(count=n()) 
```

```
## Source: local data frame [3 x 2]
## 
##         type count
## 1 Disclosure    86
## 2   Document     1
## 3  Statement     9
```

To find all statements, filter roles by type:

```r
xbrl.vars$role %>%
  filter(type == "Statement", definition == "1003000 - Statement - CONSOLIDATED BALANCE SHEETS") %>%
  select(roleId) 
```

```
##                                                             roleId
## 1 http://www.thecocacolacompany.com/role/ConsolidatedBalanceSheets
```


## Presentation hierarchy
To find out which concepts are reported on specific financial statement component, we have to search the presentation tree from the top element.


```r
library(tidyr)
library(dplyr)

# let's get the balace sheet
role_id <- "http://www.thecocacolacompany.com/role/ConsolidatedBalanceSheets"

# prepare presentation linkbase : 
# filter by role_id an convert order to numeric
pres <- 
  xbrl.vars$presentation %>%
  filter(roleId %in% role_id) %>%
  mutate(order = as.numeric(order))

# start with top element of the presentation tree
pres_df <- 
  pres %>%
  anti_join(pres, by = c("fromElementId" = "toElementId")) %>%
  select(elementId = fromElementId)

# breadth-first search
while({
  df1 <- pres_df %>%
    na.omit() %>%
    left_join( pres, by = c("elementId" = "fromElementId")) %>%
    arrange(elementId, order) %>%
    select(elementId, child = toElementId);
  nrow(df1) > 0
}) 
{
  # add each new level to data frame
  pres_df <- pres_df %>% left_join(df1, by = "elementId")
  names(pres_df) <-  c(sprintf("level%d", 1:(ncol(pres_df)-1)), "elementId")
}
# add last level as special column (the hierarchy may not be uniformly deep)
pres_df["elementId"] <- 
  apply( t(pres_df), 2, function(x){tail( x[!is.na(x)], 1)})
pres_df["elOrder"] <- 1:nrow(pres_df) 

# the final data frame structure is
str(pres_df, vec.len = 1 )
```

```
## 'data.frame':	37 obs. of  8 variables:
##  $ level1   : chr  "us-gaap_StatementOfFinancialPositionAbstract" ...
##  $ level2   : chr  "us-gaap_StatementTable" ...
##  $ level3   : chr  "us-gaap_StatementScenarioAxis" ...
##  $ level4   : chr  "us-gaap_ScenarioUnspecifiedDomain" ...
##  $ level5   : chr  NA ...
##  $ level6   : chr  NA ...
##  $ elementId: chr  "us-gaap_ScenarioUnspecifiedDomain" ...
##  $ elOrder  : int  1 2 ...
```

## Adding numbers and contexts
Elements (or _concepts_ in XBRL terminology) of the balance sheet are now gathered in data frame with presentation hierarchy levels. To see the numbers we have to join the elements with facts and contexts as in the first example.


```r
# join concepts with context, facts
pres_df_num <-
  pres_df %>%
  left_join(xbrl.vars$fact, by = "elementId") %>%
  left_join(xbrl.vars$context, by = "contextId") %>%
  filter(is.na(dimension1)) %>%
  filter(!is.na(endDate)) %>%
  select(elOrder, contains("level"), elementId, fact, decimals, endDate) %>%
  mutate( fact = as.numeric(fact) * 10^as.numeric(decimals)) %>%
  spread(endDate, fact ) %>%
  arrange(elOrder)


library(pander)

pres_df_num %>% 
  select(elementId, contains("2013"), contains("2012")) %>%
  pandoc.table(
    style = "rmarkdown",
    split.table = 200,
    justify = c("left", "right", "right"))
```



| elementId                                                                      |   2013-12-31 |   2012-12-31 |
|:-------------------------------------------------------------------------------|-------------:|-------------:|
| us-gaap_CashAndCashEquivalentsAtCarryingValue                                  |        10414 |         8442 |
| us-gaap_OtherShortTermInvestments                                              |         6707 |         5017 |
| us-gaap_CashCashEquivalentsAndShortTermInvestments                             |        17121 |        13459 |
| us-gaap_MarketableSecuritiesCurrent                                            |         3147 |         3092 |
| us-gaap_AccountsReceivableNetCurrent                                           |         4873 |         4759 |
| us-gaap_InventoryNet                                                           |         3277 |         3264 |
| us-gaap_PrepaidExpenseAndOtherAssetsCurrent                                    |         2886 |         2781 |
| us-gaap_AssetsHeldForSaleCurrent                                               |            0 |         2973 |
| us-gaap_AssetsCurrent                                                          |        31304 |        30328 |
| us-gaap_EquityMethodInvestments                                                |        10393 |         9216 |
| ko_AvailableForSaleSecuritiesAndCostMethodInvestments                          |         1119 |         1232 |
| us-gaap_OtherAssetsNoncurrent                                                  |         4661 |         3585 |
| us-gaap_PropertyPlantAndEquipmentNet                                           |        14967 |        14476 |
| us-gaap_IndefiniteLivedTrademarks                                              |         6744 |         6527 |
| us-gaap_IndefiniteLivedFranchiseRights                                         |         7415 |         7405 |
| us-gaap_Goodwill                                                               |        12312 |        12255 |
| ko_OtherIndefiniteLivedAndFiniteLivedIntangibleAssets                          |         1140 |         1150 |
| us-gaap_Assets                                                                 |        90055 |        86174 |
| us-gaap_AccountsPayableAndAccruedLiabilitiesCurrent                            |         9577 |         8680 |
| ko_LoansAndNotesPayable                                                        |        16901 |        16297 |
| us-gaap_LongTermDebtCurrent                                                    |         1024 |         1577 |
| us-gaap_AccruedIncomeTaxesCurrent                                              |          309 |          471 |
| ko_LiabilitiesHeldForSaleAtCarryingValue                                       |            0 |          796 |
| us-gaap_LiabilitiesCurrent                                                     |        27811 |        27821 |
| us-gaap_LongTermDebtNoncurrent                                                 |        19154 |        14736 |
| us-gaap_OtherLiabilitiesNoncurrent                                             |         3498 |         5468 |
| us-gaap_DeferredTaxLiabilitiesNoncurrent                                       |         6152 |         4981 |
| us-gaap_CommonStockValue                                                       |         1760 |         1760 |
| us-gaap_AdditionalPaidInCapitalCommonStock                                     |        12276 |        11379 |
| us-gaap_RetainedEarningsAccumulatedDeficit                                     |        61660 |        58045 |
| us-gaap_AccumulatedOtherComprehensiveIncomeLossNetOfTax                        |        -3432 |        -3385 |
| us-gaap_TreasuryStockValue                                                     |        39091 |        35009 |
| us-gaap_StockholdersEquity                                                     |        33173 |        32790 |
| us-gaap_MinorityInterest                                                       |          267 |          378 |
| us-gaap_StockholdersEquityIncludingPortionAttributableToNoncontrollingInterest |        33440 |        33168 |
| us-gaap_LiabilitiesAndStockholdersEquity                                       |        90055 |        86174 |

## Labels

Every concept in XBRL may have several labels (short name, description, documentation, etc.) perhaps in several languages. In presentation linkbase there is a hint (`preferredLabel`) which label should be used preferrably.
Additionally we emphasize the numbers that are computed. We use the relations from calculation linkbase. 


```r
# labels for our financial statement (role_id) in "en-US" language:
x_labels <-
  xbrl.vars$presentation %>%
  filter(roleId == role_id) %>%
  select(elementId = toElementId, labelRole = preferredLabel) %>%
  semi_join(pres_df_num, by = "elementId") %>%
  left_join(xbrl.vars$label, by = c("elementId", "labelRole")) %>%
  filter(lang == "en-US") %>%
  select(elementId, labelString)

# calculated elements in this statement component
x_calc <- xbrl.vars$calculation %>%
  filter(roleId == role_id) %>%
  select(elementId = fromElementId, calcRoleId = arcrole) %>%
  unique()

# join concepts and numbers with labels
balance_sheet_pretty <- pres_df_num %>%
  left_join(x_labels, by = "elementId") %>%
  left_join(x_calc, by = "elementId") %>%
  select(labelString, contains("2013"), contains("2012"), calcRoleId)


names(balance_sheet_pretty)[1] <- 
  "CONDENSED CONSOLIDATED BALANCE SHEETS (mio USD $)"

# rendering balance sheet
library(pander)
pandoc.table(
  balance_sheet_pretty[,1:3],
  style = "rmarkdown",
  justify = c("left", "right", "right"),
  split.table = 200,
  emphasize.strong.rows = which(!is.na(balance_sheet_pretty$calcRoleId)))
```



| CONDENSED CONSOLIDATED BALANCE SHEETS (mio USD $)                                                            |   2013-12-31 |   2012-12-31 |
|:-------------------------------------------------------------------------------------------------------------|-------------:|-------------:|
| Cash and cash equivalents                                                                                    |        10414 |         8442 |
| Short-term investments                                                                                       |         6707 |         5017 |
| **TOTAL CASH, CASH EQUIVALENTS AND SHORT-TERM INVESTMENTS**                                                  |    **17121** |    **13459** |
| Marketable securities                                                                                        |         3147 |         3092 |
| Trade accounts receivable, less allowances of $61 and $53, respectively                                      |         4873 |         4759 |
| Inventories                                                                                                  |         3277 |         3264 |
| Prepaid expenses and other assets                                                                            |         2886 |         2781 |
| Assets held for sale                                                                                         |            0 |         2973 |
| **TOTAL CURRENT ASSETS**                                                                                     |    **31304** |    **30328** |
| EQUITY METHOD INVESTMENTS                                                                                    |        10393 |         9216 |
| OTHER INVESTMENTS, PRINCIPALLY BOTTLING COMPANIES                                                            |         1119 |         1232 |
| OTHER ASSETS                                                                                                 |         4661 |         3585 |
| PROPERTY, PLANT AND EQUIPMENT - net                                                                          |        14967 |        14476 |
| TRADEMARKS WITH INDEFINITE LIVES                                                                             |         6744 |         6527 |
| BOTTLERS' FRANCHISE RIGHTS WITH INDEFINITE LIVES                                                             |         7415 |         7405 |
| GOODWILL                                                                                                     |        12312 |        12255 |
| OTHER INTANGIBLE ASSETS                                                                                      |         1140 |         1150 |
| **TOTAL ASSETS**                                                                                             |    **90055** |    **86174** |
| Accounts payable and accrued expenses                                                                        |         9577 |         8680 |
| Loans and notes payable                                                                                      |        16901 |        16297 |
| Current maturities of long-term debt                                                                         |         1024 |         1577 |
| Accrued income taxes                                                                                         |          309 |          471 |
| Liabilities held for sale                                                                                    |            0 |          796 |
| **TOTAL CURRENT LIABILITIES**                                                                                |    **27811** |    **27821** |
| LONG-TERM DEBT                                                                                               |        19154 |        14736 |
| OTHER LIABILITIES                                                                                            |         3498 |         5468 |
| DEFERRED INCOME TAXES                                                                                        |         6152 |         4981 |
| Common stock, $0.25 par value; Authorized â€” 11,200 shares; Issued â€” 7,040 and 7,040 shares, respectively |         1760 |         1760 |
| Capital surplus                                                                                              |        12276 |        11379 |
| Reinvested earnings                                                                                          |        61660 |        58045 |
| Accumulated other comprehensive income (loss)                                                                |        -3432 |        -3385 |
| Treasury stock, at cost â€” 2,638 and 2,571 shares, respectively                                             |        39091 |        35009 |
| **EQUITY ATTRIBUTABLE TO SHAREOWNERS OF THE COCA-COLA COMPANY**                                              |    **33173** |    **32790** |
| EQUITY ATTRIBUTABLE TO NONCONTROLLING INTERESTS                                                              |          267 |          378 |
| **TOTAL EQUITY**                                                                                             |    **33440** |    **33168** |
| **TOTAL LIABILITIES AND EQUITY**                                                                             |    **90055** |    **86174** |
