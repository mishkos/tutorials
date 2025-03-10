Advanced: Change MAgPIE GAMS Code
================
Florian Humpenöder (<humpenoeder@pik-potsdam.de>)
09 September, 2019

### 1 Learning objectives

MAgPIE has a modular concept. Each module (e.g. cropland) can have
several realization (e.g. dynamic and static). This tutorial shows how
to add new realizations and how to modify existing realizations.

### 2 Adding a new realization (cropland)

We want to add a new realization to the cropland module. In the MAgPIE
4.1 release the cropland module (30\_crop) has only one realization
called “endo\_jun13”. In this realization irrigation of bioenergy crops
is prohibited. We will add a new realization, based on “endo\_jun13”,
which allows for irrigation of bioenergy crops. We will call the new
realization “endo\_sep19”.

#### Create a new realization by copying an existing one

We first copy-paste the folder “endo\_jun13” and the file
“endo\_jun13.gms” (both located in “modules/30\_crop”), and rename
them to “endo\_sep19” and “endo\_sep19.gms” respectivly.

#### Include it properly via lucode::update\_modules\_embedding()

To include the new realization “endo\_sep19” properly into the GAMS code
we run a specific R command in the main folder. First navigate in your
command line to the MAgPIE main folder, then open a new R session (type
“R” followed by ENTER), and then copy-paste the following R command:
`lucode::update_modules_embedding()`

#### Do a simple modification in the new realization

The prohibition of irrigated bioenergy is specified in the file
“presolve.gms” within the “endo\_sep19” realization. We just have to
remove (or comment out) lines 8-9 from this file to allow for irrigation
of bioenergy crops.

`*vm_area.fx(j,"begr","irrigated")=0;`

`*vm_area.fx(j,"betr","irrigated")=0;`

Finally, we run `lucode::update_fulldataOutput()` to update the
automatic GAMS code declarations and output definitions.

Note: This command is not needed for this exercise. But if you change,
add or delete variables/parameters this command is of high importance to
avoid GAMS compilation errors.

#### Set the new realization in cfg and test the model

For a quick test, we simply set the new realization in the file main.gms
in line 179 (replace `endo_jun13` by `endo_sep19`). We can then check if
the model compiles correctly with `gams main.gms action=C`

For starting a model run, we would have to make a change in the config
file `config/default.cfg` in line 219 (replace `endo_jun13` by
`endo_sep19`). We could then start a model run with `Rscript start.R
-> 1: default -> 1: Direct execution`.

### 3 Changing an existing realization (cropland)

We now extend the new cropland realization `endo_sep19` by two switches
related to bioenergy, which increases the flexibility of the
realization. We add one switch for the type of bioenergy feedstock
(begr/betr), and another switch for rainfed/irrigated bioenergy
production. For this we work with dynamic sets, which are filled
depending on choices in the config file.

#### Define the switch in input.gms

We add the following lines of code to input.gms:

``` r
$setglobal c30_bioen_type  all
* options: begr, betr, all

$setglobal c30_bioen_water  rainfed
* options: rainfed, irrigated, all
```

#### Define the dynamic sets in sets.gms

We add the following lines of code to sets.gms:

``` r
   kbe30(kcr) bio energy activities
        / betr, begr /

   bioen_type_30(kbe30) dynamic set bioen type
   bioen_water_30(w) dynamic set bioen water
```

#### Fill and use the dynmic sets in presolve.gms

In presolve.gms we replace lines 8-9 with the following:

``` r
$ifthen "%c30_bioen_type%" == "all" bioen_type_30(kbe30) = yes;
$else bioen_type_30("%c30_bioen_type%") = yes;
$endif

$ifthen "%c30_bioen_water%" == "all" bioen_water_30(w) = yes;
$else bioen_water_30("%c30_bioen_water%") = yes;
$endif

*' @code
*' First, all 2nd generation bioenergy area is fixed to zero, irrespective of type and 
*' rainfed/irrigation.
vm_area.fx(j,kbe30,w)=0;
*' Second, the bounds for 2nd generation bioenergy area are released depending on 
*' the dynamic sets bioen_type_30 and bioen_water_30.
vm_area.up(j,bioen_type_30,bioen_water_30)=Inf;
*' @stop
```

#### Add to config

We can then add `c30_bioen_type` and `c30_bioen_water` to the config
file `config/default.cfg`, after line
219.

``` r
# * (c30_bioen_type): switch for type of bioenergy crops; options: begr, betr, all
cfg$gms$c30_bioen_type <- "all"               # def = "all"
# * (c30_bioen_water): switch for irrigation of bioenergy crops; options: rainfed, irrigated, all
cfg$gms$c30_bioen_water <- "rainfed"               # def = "rainfed"
```

#### Quick test

We can then check if the model compiles correctly with `gams main.gms
action=C`.
