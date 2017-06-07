## crawling_SEC_10k_Forms
R script for crawling all the 10-k forms available on the web (EDGAR website)

### Terminal and tmux:
ssh <account_ID>@<Server_IP_Address> ## Connect to a server <br />
tmux attach -t <Session Number>  ## Attach a tmux session <br />
R  # Access R <br />

### R Session:
if(!require(devtools)) install.packages("devtools") <br />
if(!require(data.table)) install.packages("data.table") <br />
if(!require(dplyr)) install.packages("dplyr") <br />
if(!require(stringi)) install.packages("stringi") <br />
if(!require(rvest)) install.packages("rvest") <br />
install_github("JasperHG90/TenK") <br />
require(TenK) ## Install and load relevant packages <br />

## Collect all SIC codes
sicLIST <- read_html("https://www.sec.gov/info/edgar/siccodes.htm") <br />
SICs <- sicLIST %>% <br />
  html_nodes("td") %>% <br />
  html_text() <br />
SICs <- SICs[19:length(SICs)] <br />
SICs <- SICs[!SICs %in% c("", " ")] <br />
SICs <- matrix(SICs, ncol = 4, byrow = T) <br />
SICs <- SICs[-nrow(SICs),] ## SIC Table <br />
SICs <- SICs[-1,1]## Removing all but the SIC codes <br />

## Collect all CIK (EDGAR company identifier) codes
i = 1 <br />
j = 1 <br />
cikVEC <- NULL <br />
pages <- seq(from = 1, to = 10000000, by = 100) <br />
for(j in 1:length(SICs)){ <br />
  for(i in 1:length(pages)){ <br />
    sec <- read_html(paste0("https://www.sec.gov/cgi-bin/browse-edgar?action=getcompany&SIC=",  <br />
    SICs[j], "&owner=include&match=&start=", pages[i], "&count=100&hidefilings=0")) <br />
    cik <- sec %>% <br />
      html_nodes("td") %>% <br />
      html_text() %>% <br />
      strsplit("\n") %>% <br />
      unlist() %>% <br />
      stri_trim_both() <br />
    if(length(cik) != 0){ <br />
      #cik <- matrix(cik[-length(cik)], ncol = 3, byrow = T) <br />
      #cik <- cik[-1,1] <br />
      cik <- cik[substr(cik,1,2) == "00"] <br />
      cikVEC <- append(cikVEC, cik) <br />
      print(paste0("j = ",j," & i = ",i , " iteration completed.")) <br />
    } <br />
    else{ <br />
      print(paste0("End of Loop for j = ", j)) <br />
      break <br />
    } <br />
  } <br />
} ## Constructing a full CIK (EDGAR company identifier) list <br />
rm(list = ls()[!(ls() %in% c('cikVEC'))])  ## Remove all else but the list of cik codes <br />

## Collect all of the URL addresses containing 10-K forms
k = 1 <br />
tenkVEC <- NULL <br />
time = Sys.time() <br />
for(k in 1:length(cikVEC)){ <br />
  TenKs <- html_session(paste0("https://www.sec.gov/cgi-bin/browse-edgar?action=getcompany&CIK=", cikVEC[k],"&type=10-k&dateb=&owner=exclude&count=100")) <br />
  tenk <- NULL <br />
  tenkUrl <- NULL <br />
  try(tenkUrl <- TenKs %>% <br />
    follow_link("Documents") %>% <br />
    html_nodes("tr td a") %>% <br />
    html_attr("href")) <br />
    
  if(!is.null(tenkUrl)){ <br />
    type <- TenKs %>% <br />
    follow_link("Documents") %>% <br />
    html_nodes("td") %>% <br />
    html_text() %>% <br />
    matrix(ncol = 5, byrow = T) <br />
    
    type <- type[,4] <br />
    tenk <- data.table(cbind(type, tenkUrl)) <br />
    tenk <- tenk[type == "10-K", tenkUrl] <br />
    tenkVEC <- append(tenkVEC, tenk) <br />
  } <br />
  print(paste0(k, " iteration completed.")) <br />
} <br />
print("End of Loop.") <br />
Sys.time() - time <br />

tenkVEC <- paste0("https://www.sec.gov",tenkVEC) <br />

## Crawling all of the 10-K forms using the exhaustive set of URLs for 10-K forms (for all of the companies registered on EDGAR)
i = 1 <br />
BD_dat <- NULL <br />
for(i in 1:length(TenKVEC)){ <br />
  res <- TenK_process(URL = tenkVEC[i], retrieve = "BD") <br />
  BD_dat <- append(res, TenK_process(URL = filings10K2013$ftp_url[1], retrieve = "BD")) <br />
} <br />

### End of Code
