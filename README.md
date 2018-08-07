
<!-- README.md is generated from README.Rmd. Please edit that file -->

# `ghql`

Using a quick query language to pull multiple api requests at once.

  - [installation](#install-ghql)
  - [tidy function](#tidy-issues)
  - [using with github](#working-with-github)

## Install ghql

``` r

remotes::install_github("ropensci/ghql")
```

``` r
library(magrittr)
```

## Query Objects

open a new query object

``` r

qry <- ghql::Query$new()
```

define the grapql query

``` r

qry$query('user_states','{
          viewer {
    repositories(privacy:PUBLIC,first: 30) {
      pageInfo {
        hasNextPage
        endCursor
      }
      edges {
        node {
          name
          pullRequests(states:[OPEN],last: 20, orderBy: {field: CREATED_AT, direction: DESC}) {
            edges {
            node {
              title
              number
              state
              createdAt
              author{
                login
              }
                }
                }
                }
          issues(states:[OPEN],last: 20, orderBy: {field: CREATED_AT, direction: DESC}) {
            edges {
            node {
              title
              number
              state
              createdAt
              author{
                login
              }
                }
                }
                }
        }
      }
    }
  }
}')
```

## tidy issues

``` r

tidy_issues <- function(x){
  
  x1 <- jsonlite::fromJSON(x)
  
  edges <- purrr::transpose(x1$data$viewer$repositories$edges)%>%
    dplyr::as_tibble()
  
  open_edges <- edges%>%
    dplyr::select(name,pullRequests)%>%
    tidyr::unnest()%>%
    dplyr::rename(open_pullRequests = edges)%>%
    dplyr::left_join(
      edges%>%
        dplyr::select(name,issues)%>%
        tidyr::unnest()%>%
        dplyr::rename(open_issues = edges),
      by='name'
    )
 
  PR <- purrr::transpose(open_edges$open_pullRequests)%>%
    purrr::flatten()%>%
    purrr::set_names(open_edges$name)%>%
    purrr::discard(.p = is.null)%>%
    purrr::map_df(.f=function(x) {
      x$author <- purrr::flatten_chr(x$author)
      x},.id = 'repository')%>%
    dplyr::mutate(type = 'pull_request')%>%
    dplyr::select(-state)
  
  ISSUES <- purrr::transpose(open_edges$open_issues)%>%
    purrr::flatten()%>%
    purrr::set_names(open_edges$name)%>%
    purrr::discard(.p = is.null)%>%
    purrr::map_df(.f=function(x) {
      x$author <- purrr::flatten_chr(x$author)
      x},.id = 'repository')%>%
    dplyr::mutate(type = 'issue')%>%
    dplyr::select(-state)

  output <- dplyr::bind_rows(PR,ISSUES)
  
  output$createdAt = as.POSIXct(output$createdAt, "UTC", "%Y-%m-%dT%H:%M:%S")
  output$days_passed = as.numeric(difftime(Sys.time(),output$createdAt,units = 'days'))
  
  output
  
}
```

## Working with Github

load gh client with GITHUB\_PAT

``` r
  
    cli_gh <- ghql::GraphqlClient$new(
      url = "https://api.github.com/graphql",
      headers = httr::add_headers(Authorization = sprintf("Bearer %s", Sys.getenv("GITHUB_PAT")))
    )
```

load the schema

``` r
  
cli_gh$load_schema()
```

execute the graphql query

``` r
  
x_gh <- cli_gh$exec(qry$queries$user_states)       
```

tidy up

``` r
  
output <- tidy_issues(x_gh)
```

``` r
output%>%
  dplyr::glimpse()
#> Observations: 52
#> Variables: 7
#> $ repository  <chr> "slackr", "slackr", "slackr", "heatmaply", "slackr...
#> $ title       <chr> "typos", "htmlslackr", "issue #68", "Cellnote dots...
#> $ number      <int> 73, 72, 71, 172, 74, 70, 69, 68, 67, 62, 59, 58, 5...
#> $ createdAt   <dttm> 2018-04-12 15:42:47, 2018-03-27 18:08:54, 2018-03...
#> $ author      <chr> "MichaelChirico", "overmar", "overmar", "Alanocall...
#> $ type        <chr> "pull_request", "pull_request", "pull_request", "p...
#> $ days_passed <dbl> 117.2904, 133.1889, 133.1986, 116.0946, 112.2774, ...
```

<details>

<summary> <span title="Click to Expand"> Pull Request Table </span>
</summary>

``` r
output%>%
  dplyr::filter(type=='pull_request')%>%
  knitr::kable()
```

| repository | title         | number | createdAt           | author         | type          | days\_passed |
| :--------- | :------------ | -----: | :------------------ | :------------- | :------------ | -----------: |
| slackr     | typos         |     73 | 2018-04-12 15:42:47 | MichaelChirico | pull\_request |     117.2904 |
| slackr     | htmlslackr    |     72 | 2018-03-27 18:08:54 | overmar        | pull\_request |     133.1889 |
| slackr     | issue \#68    |     71 | 2018-03-27 17:54:52 | overmar        | pull\_request |     133.1986 |
| heatmaply  | Cellnote dots |    172 | 2018-04-13 20:24:36 | Alanocallaghan | pull\_request |     116.0946 |

</details>

<details>

<summary> <span title="Click to Expand"> Issues Table </span> </summary>

``` r
output%>%
  dplyr::filter(type=='issue')%>%
  knitr::kable()
```

| repository     | title                                                                                      | number | createdAt           | author         | type  | days\_passed |
| :------------- | :----------------------------------------------------------------------------------------- | -----: | :------------------ | :------------- | :---- | -----------: |
| slackr         | Publish latest version to CRAN?                                                            |     74 | 2018-04-17 16:01:26 | maxscheiber    | issue |   112.277399 |
| slackr         | slackr not sending output to private groups                                                |     70 | 2018-03-04 17:13:10 | deann88        | issue |   156.227585 |
| slackr         | Not resolved - “icon\_emoji bug?” (in closed)                                              |     69 | 2018-01-17 19:25:55 | nataliagalant  | issue |   202.135397 |
| slackr         | slackr\_upload fails with .gif                                                             |     68 | 2017-11-14 14:44:49 | tomauer        | issue |   266.330605 |
| slackr         | Add a note to documentation explaining where you get an API token                          |     67 | 2017-11-07 20:01:45 | smithjd        | issue |   273.110513 |
| slackr         | simple way to add team members to app                                                      |     62 | 2017-09-22 18:25:46 | yonicd         | issue |   319.177168 |
| slackr         | can slackr fail noisily if Slack responds with FALSE ok status?                            |     59 | 2017-06-08 21:59:00 | ronnyli        | issue |   425.029089 |
| slackr         | slackr(str(iris)) Error in fix.by(by.x, x) : ‘by’ must specify a uniquely valid column     |     58 | 2017-06-08 08:28:37 | vipula86       | issue |   425.591856 |
| slackr         | Can’t post to channel with period contained in name                                        |     56 | 2017-03-27 08:18:48 | samgilbert1983 | issue |   498.598673 |
| slackr         | slackr\_bot can not handle " " inside expressions                                          |     54 | 2017-02-14 10:45:17 | Javdat         | issue |   539.496948 |
| slackr         | Not all parameters being passed to slacker POST                                            |     53 | 2017-01-24 18:42:05 | calz1          | issue |   560.165837 |
| slackr         | ‘id’ column not found in lhs                                                               |     52 | 2017-01-16 03:29:47 | isomorphisms   | issue |   568.799379 |
| slackr         | slackr() not sending messages to groups                                                    |     50 | 2016-12-15 09:38:40 | rontomer       | issue |   600.543210 |
| slackr         | Error using slack or slack\_msg                                                            |     47 | 2016-11-08 22:18:29 | mst86          | issue |   637.015559 |
| slackr         | `slackr_chtrans` does not find private channels if they start with `#`                     |     45 | 2016-08-30 11:04:40 | anniejw6       | issue |   707.483487 |
| slackr         | asynchronous slackr requests                                                               |     44 | 2016-07-25 14:07:38 | Popsch         | issue |   743.356427 |
| slackr         | korean encoding problem                                                                    |     38 | 2016-06-07 07:03:24 | swoosh22       | issue |   791.651034 |
| slackr         | SlackrUpload not including all contents from CSV file                                      |     35 | 2016-05-04 21:49:11 | raybuhr        | issue |   825.035906 |
| slackr         | Ability to listen to messages sent from slack                                              |     29 | 2016-02-13 19:07:50 | jemus42        | issue |   906.147955 |
| slackr         | pass graphics devices as parameter to dev.slackr                                           |     14 | 2015-05-04 21:34:51 | wagnerpinheiro | issue |  1191.045860 |
| taseR          | Can I ask a few questions regarding this repo?                                             |      1 | 2018-01-02 12:18:25 | koren88i       | issue |   217.432272 |
| heatmaply      | How much can the file size be reduced?                                                     |     79 | 2017-06-12 18:27:22 | talgalili      | issue |   421.176057 |
| heatmaply      | row/column label overplotting                                                              |     78 | 2017-06-12 09:21:44 | Alanocallaghan | issue |   421.554969 |
| heatmaply      | Allow heatmap to be saved as a high resolution svg (or png/ etc.)                          |     77 | 2017-06-10 08:12:48 | talgalili      | issue |   423.602839 |
| heatmaply      | Hover with cellnote issue                                                                  |     76 | 2017-06-01 11:04:14 | chrbac         | issue |   432.483788 |
| heatmaply      | Getting an error: The length of the widths argument must be equal to the number of columns |     72 | 2017-05-29 11:27:23 | ndrubins       | issue |   435.467712 |
| heatmaply      | Interactively highlight rows and/or columns                                                |     71 | 2017-05-22 08:54:08 | andbiy         | issue |   442.574136 |
| heatmaply      | ‘Pan’ mode allows user to move outside the plot margins                                    |     70 | 2017-05-22 08:47:51 | andbiy         | issue |   442.578499 |
| heatmaply      | Passing ggdend objects to heatmapr rather than dendrograms                                 |     65 | 2017-05-08 22:58:53 | ndrubins       | issue |   455.987504 |
| heatmaply      | Automatic margins                                                                          |     62 | 2017-04-13 11:25:52 | talgalili      | issue |   481.468765 |
| heatmaply      | Implement hclust\_method=NA                                                                |     58 | 2017-04-12 08:45:15 | talgalili      | issue |   482.580305 |
| heatmaply      | seriation for dendrogram=“none”                                                            |     57 | 2017-04-12 08:43:58 | talgalili      | issue |   482.581196 |
| heatmaply      | Get grid\_gap to work when RowSideColors is used                                           |     53 | 2017-04-11 08:04:01 | talgalili      | issue |   483.608939 |
| heatmaply      | Encapsulate the creation of dendrograms                                                    |     44 | 2017-04-03 20:02:05 | talgalili      | issue |   491.110281 |
| heatmaply      | Legends for side colour plots                                                              |     35 | 2017-03-08 00:38:52 | Alanocallaghan | issue |   517.918071 |
| heatmaply      | Buttons in plot window crashes RStudio                                                     |     17 | 2016-07-28 14:06:48 | DaveEvenden    | issue |   740.357006 |
| heatmaply      | Row labels missing when row\_dend\_left = TRUE                                             |     14 | 2016-06-24 23:52:43 | kbeck527       | issue |   773.950119 |
| heatmaply      | More general correlations without assuming linearity                                       |     13 | 2016-05-31 23:17:23 | hdvinod        | issue |   797.974656 |
| heatmaply      | na.value doesn’t work                                                                      |     11 | 2016-05-26 18:37:23 | talgalili      | issue |   803.169101 |
| heatmaply      | Possibility to supply cellnote as a character matrix                                       |     10 | 2016-05-26 14:09:49 | stanstrup      | issue |   803.354911 |
| heatmaply      | Possibility for non-unique row/col names?                                                  |      9 | 2016-05-26 10:46:34 | stanstrup      | issue |   803.496057 |
| shinyHeatmaply | Interactively drag/drop and remove columns/lines                                           |     21 | 2017-05-22 08:56:59 | andbiy         | issue |   442.572156 |
| shinyHeatmaply | Add option to control plot\_method                                                         |     20 | 2017-04-12 07:24:00 | talgalili      | issue |   482.636728 |
| shinyHeatmaply | Add heatmaply height as a parameter in layout                                              |     14 | 2017-03-23 09:50:02 | talgalili      | issue |   502.535316 |
| slickR         | Slow render speeds                                                                         |     18 | 2018-06-26 04:27:32 | RossPitman     | issue |    42.759274 |
| slickR         | Error when using slickR in shiny                                                           |     16 | 2018-04-03 22:16:41 | cegbuna        | issue |   126.016809 |
| sinew          | pretty\_namespace corrupting text                                                          |     20 | 2018-08-05 01:20:09 | dpastoor       | issue |     2.889402 |
| sinew          | RcppExports.R                                                                              |     17 | 2017-12-19 17:42:23 | daniel258      | issue |   231.207295 |

</details>
