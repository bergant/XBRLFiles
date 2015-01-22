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

inst <- "~/R/XBRLTest/Data/edgar/ko-20131231.xml"
#inst <- "http://edgar.sec.gov/Archives/edgar/data/21344/000002134414000008/ko-20131231.xml"
#options(stringsAsFactors = FALSE)
#xbrl.vars <- xbrlDoAll(inst, cache.dir = "XBRLcache", prefix.out = NULL)
load("~/R/XBRLTest/xbrl.vars_cocacola.RData")
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
    justify = c("left", "right", "right"))
```



| elementId                                                                      |
|:-------------------------------------------------------------------------------|
| us-gaap_CashAndCashEquivalentsAtCarryingValue                                  |
| us-gaap_OtherShortTermInvestments                                              |
| us-gaap_CashCashEquivalentsAndShortTermInvestments                             |
| us-gaap_MarketableSecuritiesCurrent                                            |
| us-gaap_AccountsReceivableNetCurrent                                           |
| us-gaap_InventoryNet                                                           |
| us-gaap_PrepaidExpenseAndOtherAssetsCurrent                                    |
| us-gaap_AssetsHeldForSaleCurrent                                               |
| us-gaap_AssetsCurrent                                                          |
| us-gaap_EquityMethodInvestments                                                |
| ko_AvailableForSaleSecuritiesAndCostMethodInvestments                          |
| us-gaap_OtherAssetsNoncurrent                                                  |
| us-gaap_PropertyPlantAndEquipmentNet                                           |
| us-gaap_IndefiniteLivedTrademarks                                              |
| us-gaap_IndefiniteLivedFranchiseRights                                         |
| us-gaap_Goodwill                                                               |
| ko_OtherIndefiniteLivedAndFiniteLivedIntangibleAssets                          |
| us-gaap_Assets                                                                 |
| us-gaap_AccountsPayableAndAccruedLiabilitiesCurrent                            |
| ko_LoansAndNotesPayable                                                        |
| us-gaap_LongTermDebtCurrent                                                    |
| us-gaap_AccruedIncomeTaxesCurrent                                              |
| ko_LiabilitiesHeldForSaleAtCarryingValue                                       |
| us-gaap_LiabilitiesCurrent                                                     |
| us-gaap_LongTermDebtNoncurrent                                                 |
| us-gaap_OtherLiabilitiesNoncurrent                                             |
| us-gaap_DeferredTaxLiabilitiesNoncurrent                                       |
| us-gaap_CommonStockValue                                                       |
| us-gaap_AdditionalPaidInCapitalCommonStock                                     |
| us-gaap_RetainedEarningsAccumulatedDeficit                                     |
| us-gaap_AccumulatedOtherComprehensiveIncomeLossNetOfTax                        |
| us-gaap_TreasuryStockValue                                                     |
| us-gaap_StockholdersEquity                                                     |
| us-gaap_MinorityInterest                                                       |
| us-gaap_StockholdersEquityIncludingPortionAttributableToNoncontrollingInterest |
| us-gaap_LiabilitiesAndStockholdersEquity                                       |

Table: Table continues below

 

|   2013-12-31 |   2012-12-31 |
|-------------:|-------------:|
|        10414 |         8442 |
|         6707 |         5017 |
|        17121 |        13459 |
|         3147 |         3092 |
|         4873 |         4759 |
|         3277 |         3264 |
|         2886 |         2781 |
|            0 |         2973 |
|        31304 |        30328 |
|        10393 |         9216 |
|         1119 |         1232 |
|         4661 |         3585 |
|        14967 |        14476 |
|         6744 |         6527 |
|         7415 |         7405 |
|        12312 |        12255 |
|         1140 |         1150 |
|        90055 |        86174 |
|         9577 |         8680 |
|        16901 |        16297 |
|         1024 |         1577 |
|          309 |          471 |
|            0 |          796 |
|        27811 |        27821 |
|        19154 |        14736 |
|         3498 |         5468 |
|         6152 |         4981 |
|         1760 |         1760 |
|        12276 |        11379 |
|        61660 |        58045 |
|        -3432 |        -3385 |
|        39091 |        35009 |
|        33173 |        32790 |
|          267 |          378 |
|        33440 |        33168 |
|        90055 |        86174 |

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
  decimal.mark = ',',
  emphasize.strong.rows = which(!is.na(balance_sheet_pretty$calcRoleId)))
```



| CONDENSED CONSOLIDATED BALANCE SHEETS (mio USD $)                                                            |
|:-------------------------------------------------------------------------------------------------------------|
| Cash and cash equivalents                                                                                    |
| Short-term investments                                                                                       |
| **TOTAL CASH, CASH EQUIVALENTS AND SHORT-TERM INVESTMENTS**                                                  |
| Marketable securities                                                                                        |
| Trade accounts receivable, less allowances of $61 and $53, respectively                                      |
| Inventories                                                                                                  |
| Prepaid expenses and other assets                                                                            |
| Assets held for sale                                                                                         |
| **TOTAL CURRENT ASSETS**                                                                                     |
| EQUITY METHOD INVESTMENTS                                                                                    |
| OTHER INVESTMENTS, PRINCIPALLY BOTTLING COMPANIES                                                            |
| OTHER ASSETS                                                                                                 |
| PROPERTY, PLANT AND EQUIPMENT - net                                                                          |
| TRADEMARKS WITH INDEFINITE LIVES                                                                             |
| BOTTLERS' FRANCHISE RIGHTS WITH INDEFINITE LIVES                                                             |
| GOODWILL                                                                                                     |
| OTHER INTANGIBLE ASSETS                                                                                      |
| **TOTAL ASSETS**                                                                                             |
| Accounts payable and accrued expenses                                                                        |
| Loans and notes payable                                                                                      |
| Current maturities of long-term debt                                                                         |
| Accrued income taxes                                                                                         |
| Liabilities held for sale                                                                                    |
| **TOTAL CURRENT LIABILITIES**                                                                                |
| LONG-TERM DEBT                                                                                               |
| OTHER LIABILITIES                                                                                            |
| DEFERRED INCOME TAXES                                                                                        |
| Common stock, $0.25 par value; Authorized â€” 11,200 shares; Issued â€” 7,040 and 7,040 shares, respectively |
| Capital surplus                                                                                              |
| Reinvested earnings                                                                                          |
| Accumulated other comprehensive income (loss)                                                                |
| Treasury stock, at cost â€” 2,638 and 2,571 shares, respectively                                             |
| **EQUITY ATTRIBUTABLE TO SHAREOWNERS OF THE COCA-COLA COMPANY**                                              |
| EQUITY ATTRIBUTABLE TO NONCONTROLLING INTERESTS                                                              |
| **TOTAL EQUITY**                                                                                             |
| **TOTAL LIABILITIES AND EQUITY**                                                                             |

Table: Table continues below

 

|   2013-12-31 |   2012-12-31 |
|-------------:|-------------:|
|        10414 |         8442 |
|         6707 |         5017 |
|    **17121** |    **13459** |
|         3147 |         3092 |
|         4873 |         4759 |
|         3277 |         3264 |
|         2886 |         2781 |
|            0 |         2973 |
|    **31304** |    **30328** |
|        10393 |         9216 |
|         1119 |         1232 |
|         4661 |         3585 |
|        14967 |        14476 |
|         6744 |         6527 |
|         7415 |         7405 |
|        12312 |        12255 |
|         1140 |         1150 |
|    **90055** |    **86174** |
|         9577 |         8680 |
|        16901 |        16297 |
|         1024 |         1577 |
|          309 |          471 |
|            0 |          796 |
|    **27811** |    **27821** |
|        19154 |        14736 |
|         3498 |         5468 |
|         6152 |         4981 |
|         1760 |         1760 |
|        12276 |        11379 |
|        61660 |        58045 |
|        -3432 |        -3385 |
|        39091 |        35009 |
|    **33173** |    **32790** |
|          267 |          378 |
|    **33440** |    **33168** |
|    **90055** |    **86174** |
