## crawling_SEC_10k_Forms
R script for crawling all the 10-k forms available on the web (EDGAR website)

### Terminal and tmux:
```r
ssh <account_ID>@<Server_IP_Address>  ## Connect to a server
tmux attach -t <Session Number>  ## Attach a tmux session
R  ## Access R
```

### R Session:
```r
load("./170607_SEC_Crawling.RData")
if(!require(devtools)) install.packages("devtools")
if(!require(data.table)) install.packages("data.table")
if(!require(dplyr)) install.packages("dplyr")
if(!require(stringi)) install.packages("stringi")
if(!require(rvest)) install.packages("rvest")
install_github("JasperHG90/TenK")
require(TenK)
require(stringi)
```

## Collect all SIC codes
```r
sicLIST <- read_html("https://www.sec.gov/info/edgar/siccodes.htm")
SICs <- sicLIST %>%
  html_nodes("td") %>%
  html_text()
SICs <- SICs[19:length(SICs)]
SICs <- SICs[!SICs %in% c("", " ")]
SICs <- matrix(SICs, ncol = 4, byrow = T)
SICs <- SICs[-nrow(SICs),] ## SIC Table
SICs <- SICs[-1,1] ## Removing all but the SIC codes
```

## Collect all CIK (EDGAR company identifier) codes
```r
i = 1
j = 1
cikVEC <- NULL
pages <- seq(from = 1, to = 10000000, by = 100)
for(j in 1:length(SICs)){
  for(i in 1:length(pages)){
    sec <- read_html(paste0("https://www.sec.gov/cgi-bin/browse-edgar?action=getcompany&SIC=",
    SICs[j], "&owner=include&match=&start=", pages[i], "&count=100&hidefilings=0"))
    cik <- sec %>%
      html_nodes("td") %>%
      html_text() %>%
      strsplit("\n") %>%
      unlist() %>%
      stri_trim_both()
    if(length(cik) != 0){
      #cik <- matrix(cik[-length(cik)], ncol = 3, byrow = T)
      #cik <- cik[-1,1]
      cik <- cik[substr(cik,1,2) == "00"]
      cikVEC <- append(cikVEC, cik)
      print(paste0("j = ",j," & i = ",i , " iteration completed."))
    }
    else{
      print(paste0("End of Loop for j = ", j))
      break
    }
  }
} ## Constructing a full CIK (EDGAR company identifier) list
rm(list = ls()[!(ls() %in% c('cikVEC'))])
```

## Collect all of the URL addresses containing 10-K forms
```r
k = 1
tenkVEC <- NULL
time = Sys.time()
for(k in 1:length(cikVEC)){
  TenKs <- html_session(paste0("https://www.sec.gov/cgi-bin/browse-edgar?action=getcompany&CIK=", cikVEC[k],"&type=10-k&dateb=&owner=exclude&count=100"))
  tenk <- NULL
  tenkUrl <- NULL
  try(tenkUrl <- TenKs %>%
    follow_link("Documents") %>%
    html_nodes("tr td a") %>%
    html_attr("href"))
    
  if(!is.null(tenkUrl)){
    type <- TenKs %>%
    follow_link("Documents") %>%
    html_nodes("td") %>%
    html_text() %>%
    matrix(ncol = 5, byrow = T)
    
    type <- type[,4]
    tenk <- data.table(cbind(type, tenkUrl))
    tenk <- tenk[type == "10-K", tenkUrl]
    tenkVEC <- append(tenkVEC, tenk)
  }
  print(paste0(k, " iteration completed."))
}
print("End of Loop.")
Sys.time() - time

tenkVEC <- paste0("https://www.sec.gov",tenkVEC)
```

## Crawling all of the 10-K forms using the exhaustive set of URLs for 10-K forms (for all of the companies registered on EDGAR)
```r
i = 1
BD_dat <- NULL
for(i in 1:length(TenKVEC)){
  res <- TenK_process(URL = tenkVEC[i], retrieve = "BD")
  BD_dat <- append(res, TenK_process(URL = filings10K2013$ftp_url[1], retrieve = "BD"))
}
```
### End of Code
