
# tarchetypes <img src='man/figures/logo.png' align="right" height="139"/>

[![ropensci](https://badges.ropensci.org/401_status.svg)](https://github.com/ropensci/software-review/issues/401)
[![zenodo](https://zenodo.org/badge/282774543.svg)](https://zenodo.org/badge/latestdoi/282774543)
[![R
Targetopia](https://img.shields.io/badge/R_Targetopia-member-blue?style=flat&labelColor=gray)](https://wlandau.github.io/targetopia/)
[![CRAN](https://www.r-pkg.org/badges/version/tarchetypes)](https://CRAN.R-project.org/package=tarchetypes)
[![status](https://www.repostatus.org/badges/latest/active.svg)](https://www.repostatus.org/#active)
[![check](https://github.com/ropensci/tarchetypes/workflows/check/badge.svg)](https://github.com/ropensci/tarchetypes/actions?query=workflow%3Acheck)
[![codecov](https://codecov.io/gh/ropensci/tarchetypes/branch/main/graph/badge.svg?token=3T5DlLwUVl)](https://app.codecov.io/gh/ropensci/tarchetypes)
[![lint](https://github.com/ropensci/tarchetypes/workflows/lint/badge.svg)](https://github.com/ropensci/tarchetypes/actions?query=workflow%3Alint)

The `tarchetypes` R package is a collection of target and pipeline
archetypes for the [`targets`](https://github.com/ropensci/targets)
package. These archetypes express complicated pipelines with concise
syntax, which enhances readability and thus reproducibility. Archetypes
are possible because of the flexible metaprogramming capabilities of
[`targets`](https://github.com/ropensci/targets). In
[`targets`](https://github.com/ropensci/targets), one can define a
target as an object outside the central pipeline, and the
[`tar_target_raw()`](https://docs.ropensci.org/targets/reference/tar_target_raw.html)
function completely avoids non-standard evaluation. That means anyone
can write their own niche interfaces for specialized projects.
`tarchetypes` aims to include the most common and versatile archetypes
and usage patterns.

## Grouped data frames

`tarchetypes` has functions for easy dynamic branching over subsets of
data frames:

- `tar_group_by()`: define row groups using `dplyr::group_by()`
  semantics.
- `tar_group_select()`: define row groups using `tidyselect` semantics.
- `tar_group_count()`: define a given number row groups.
- `tar_group_size()`: define row groups of a given size.

If you define a target with one of these functions, all downstream
dynamic targets will automatically branch over the row groups.

``` r
# _targets.R file:
library(targets)
library(tarchetypes)
produce_data <- function() {
  expand.grid(var1 = c("a", "b"), var2 = c("c", "d"), rep = c(1, 2, 3))
}
list(
  tar_group_by(data, produce_data(), var1, var2),
  tar_target(group, data, pattern = map(data))
)
```

``` r
# R console:
library(targets)
tar_make()
#> • start target data
#> • built target data [0.023 seconds]
#> • start branch group_b3d7d010
#> • built branch group_b3d7d010 [0 seconds]
#> • start branch group_6a76c5c0
#> • built branch group_6a76c5c0 [0.001 seconds]
#> • start branch group_164b16bf
#> • built branch group_164b16bf [0 seconds]
#> • start branch group_f5aae602
#> • built branch group_f5aae602 [0 seconds]
#> • built pattern group
#> • end pipeline [0.155 seconds]

# First row group:
tar_read(group, branches = 1)
#> # A tibble: 3 × 4
#>   var1  var2    rep tar_group
#>   <fct> <fct> <dbl>     <int>
#> 1 a     c         1         1
#> 2 a     c         2         1
#> 3 a     c         3         1

# Second row group:
tar_read(group, branches = 2)
#> # A tibble: 3 × 4
#>   var1  var2    rep tar_group
#>   <fct> <fct> <dbl>     <int>
#> 1 a     d         1         2
#> 2 a     d         2         2
#> 3 a     d         3         2
```

## Literate programming

Consider the following R Markdown report.

    ---
    title: report
    output: html_document
    ---

    ```{r}
    library(targets)
    tar_read(dataset)
    ```

We want to define a target to render the report. And because the report
calls `tar_read(dataset)`, this target needs to depend on `dataset`.
Without `tarchetypes`, it is cumbersome to set up the pipeline
correctly.

``` r
# _targets.R
library(targets)
list(
  tar_target(dataset, data.frame(x = letters)),
  tar_target(
    report, {
      # Explicitly mention the symbol `dataset`.
      list(dataset)
      # Return relative paths to keep the project portable.
      fs::path_rel(
        # Need to return/track all input/output files.
        c( 
          rmarkdown::render(
            input = "report.Rmd",
            # Always run from the project root
            # so the report can find _targets/.
            knit_root_dir = getwd(),
            quiet = TRUE
          ),
          "report.Rmd"
        )
      )
    },
    # Track the input and output files.
    format = "file",
    # Avoid building small reports on HPC.
    deployment = "main"
  )
)
```

With `tarchetypes`, we can simplify the pipeline with the `tar_render()`
archetype.

``` r
# _targets.R
library(targets)
library(tarchetypes)
list(
  tar_target(dataset, data.frame(x = letters)),
  tar_render(report, "report.Rmd")
)
```

Above, `tar_render()` scans code chunks for mentions of targets in
`tar_load()` and `tar_read()`, and it enforces the dependency
relationships it finds. In our case, it reads `report.Rmd` and then
forces `report` to depend on `dataset`. That way, `tar_make()` always
processes `dataset` before `report`, and it automatically reruns
`report.Rmd` whenever `dataset` changes.

## Alternative pipeline syntax

[`tar_plan()`](https://docs.ropensci.org/tarchetypes/reference/tar_plan.html)
is a drop-in replacement for
[`drake_plan()`](https://docs.ropensci.org/drake/reference/drake_plan.html)
in the [`targets`](https://github.com/ropensci/targets) ecosystem. It
lets users write targets as name/command pairs without having to call
[`tar_target()`](https://docs.ropensci.org/targets/reference/tar_target.html).

``` r
tar_plan(
  tar_file(raw_data_file, "data/raw_data.csv", format = "file"),
  # Simple drake-like syntax:
  raw_data = read_csv(raw_data_file, col_types = cols()),
  data =raw_data %>%
    mutate(Ozone = replace_na(Ozone, mean(Ozone, na.rm = TRUE))),
  hist = create_plot(data),
  fit = biglm(Ozone ~ Wind + Temp, data),
  # Needs tar_render() because it is a target archetype:
  tar_render(report, "report.Rmd")
)
```

## Installation

| Type        | Source   | Command                                                               |
|-------------|----------|-----------------------------------------------------------------------|
| Release     | CRAN     | `install.packages("tarchetypes")`                                     |
| Development | GitHub   | `remotes::install_github("ropensci/tarchetypes")`                     |
| Development | rOpenSci | `install.packages("tarchetypes", repos = "https://dev.ropensci.org")` |

## Documentation

For specific documentation on `tarchetypes`, including the help files of
all user-side functions, please visit the [reference
website](https://docs.ropensci.org/tarchetypes/). For documentation on
[`targets`](https://github.com/ropensci/targets) in general, please
visit the [`targets` reference
website](https://docs.ropensci.org/targets/). Many of the linked
resources use `tarchetypes` functions such as
[`tar_render()`](https://docs.ropensci.org/tarchetypes/reference/tar_render.html).

## Help

Please read the [help
guide](https://books.ropensci.org/targets/help.html) to learn how best
to ask for help using `targets` and `tarchetypes`.

## Code of conduct

Please note that this package is released with a [Contributor Code of
Conduct](https://ropensci.org/code-of-conduct/).

## Citation

``` r
citation("tarchetypes")
#> 
#> To cite tarchetypes in publications use:
#> 
#>   William Michael Landau (2021). tarchetypes: Archetypes for Targets.
#>   https://docs.ropensci.org/tarchetypes/,
#>   https://github.com/ropensci/tarchetypes.
#> 
#> A BibTeX entry for LaTeX users is
#> 
#>   @Manual{,
#>     title = {tarchetypes: Archetypes for Targets},
#>     author = {William Michael Landau},
#>     year = {2021},
#>     note = {{https://docs.ropensci.org/tarchetypes/, https://github.com/ropensci/tarchetypes}},
#>   }
```
