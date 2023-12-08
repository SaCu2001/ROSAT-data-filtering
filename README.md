# ROSAT-data-filtering
Applying filters to the Second ROSAT All-Sky Survey Point Source Catalog using Python and TOPCAT, in order to generate a list of potential candidates for X-ray Dim Isolated Neutron Stars (XDINs) in the ROSAT data

1) Downloading the ROSAT catalog

Access the ROSAT catalog using the following link: https://heasarc.gsfc.nasa.gov/W3Browse/rosat/rass2rxs.html.

You can choose which columns to download by clicking on the 'browse this table...' button in the top left corner of the page. Here, keep all the default selected columns, but also tick the following columns too: hardness_ratio_1_error, hardness_ratio_2_error, bb_flux, x_pixel, x_pixel_error, y_pixel, y_pixel_error.

Scroll down to the bottom of the page to where it says 'Do you want to change your current query settings?', and change the 'Limit Results to' section to 'No Limit', and the 'Ouput Format' to be a FITS file.

Click Start Search and download the data file which should contain 135118 objects.

2) Applying initial filters in Python

Open a new Python document, and write the following code:

    from astropy.io import fits
    from astropy.table import Table
    import numpy as np

Open and access the FITS file downloaded previously, you can do this as follows:

    fits_file_path = "[insert the file directory here]"
    initial_data = fits.open(fits_file_path)
    data = initial_data[1].data

2.1) Remove anything with a Hardness Ratio greater than 0. 

This step is taken due to the fact that only 'soft' x-ray emitters are wanted since this is a property of an X-ray Dim Isolated Neutron Star. Do this for both the hardness_ratio_1 and hardness_ratio_2 columns. After these steps there should be 76077 objects remaining. You can do this as follows by selecting these columns from the imported data:

    column1 = 'HARDNESS_RATIO_1'
    column2 = 'HARDNESS_RATIO_1_ERROR'
    max_value = 0
    filtered_data = data[(data[column1] - data[column2]) <= max_value]

2.2) Remove anything with a positional uncertainty greater than 15 arcseconds.

An uncertainty of 15 arcseconds is chosen due to the fact that the 7 known XDINS all have low positional uncertainties, so the assumption is made that this is the general case. Firstly, convert the X and Y pixel uncertainties by 45 in order to convert from pixels to arcseconds, then combine them into a positional uncertainty by adding these X and Y uncertainties in quadrature. Additionally, save this new positional uncertainty as a new column of data in the FITS file. After this step there should be 28748 objects remaining. You can do this step as follows:

    column1 = 'X_PIXEL_ERROR'
    column2 = 'Y_PIXEL_ERROR'
    max_difference = 15

    converted_column1 = filtered_data[column1]*45
    converted_column2 = filtered_data[column2]*45
    position_uncertainty = np.sqrt((converted_column1**2)+(converted_column2**2))

    filtered_data = Table(filtered_data)
    filtered_data['POSITIONAL_UNCERTAINTY'] = position_uncertainty

    filtered_data = filtered_data[(position_uncertainty) <= max_difference]

2.3) Remove anything with a log x-ray to optical flux ratio less than 2.5. 

This step is taken due to the fact that of the known XDINS (Magnificent 7), the lowest log optical-to-x-ray flux ratio is 2.63, so an appropriate cutoff value of 2.5 is chosen. When performing this step, a minute correction within the log(x-ray flux) is required so not to obtain an error code within Python due to the fact that several of the x-ray flux inputs are zero - this correction is small enough to have no impact on the final output data. The visual magnitude to be used is the visual magnitude of the SDSS all-sky survey. After this step there should be 9358 objects remaining in the data. You can do this step as follows:

    column = 'BB_FLUX'
    max_difference = 2.5
    Visual_mag = 22.5
    
    flux_ratio = np.log10(filtered_data[column]+(1*(10**-100)))+(Visual_mag/2.5)+5.37

    filtered_data = filtered_data[(flux_ratio) >= max_difference]

This filtered data set can now be exported for further analysis in TOPCAT. You can export it as a new FITS file as follows:

    output_file = "[Insert new file directory here]"
    fits.writeto(output_file, np.asarray(filtered_data), overwrite=True)

3) Applying filters in TOPCAT - removing known objects from other all-sky surveys.

Download the outputted Python FITS file onto TOPCAT by selecting 'Open new table' in the top left corner, then 'System browser', and select the FITS file.

![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/97892d8c-ff75-41de-ac9c-ff99569c64fb)

3.1) Remove any object within 3 arcminutes of an SDSS DR16 source. 

Select 'Sky crossmatch' (the blue cross on the top toolbar). Select SDSS DR16 as the remote table, and the imported FITS file as the local table, with 'RA' and 'DEC' selected in units of degrees. In the 'Match Parameters' section, set a radius of 3 arcminutes, with 'Best' selected as the mode. All other variables leave as default. Select 'Go' and TOPCAT will filter out data within these established parameters. A new table will be generated on the TOPCAT page on the left-hand side containing 4022 objects.

![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/206f7d41-6731-408f-988f-8d7e72daf2bf)

3.2) Remove anything within 2x its positional uncertainty of an SDSS DR16 source. 

Select the matching rows button (two red matches on the top toolbar). In the 'Match criteria' section, select the 'Sky with errors' algorithm and set the scale to 10 arcminutes. Select the table produced in part 3.1) in both the 'Table 1' and 'Table 2' sections. In Table 1, select the RA and DEC columns with units of degrees, and the positional uncertainty column with units of arcseconds. In Table 2, select the RA_ICRS and DE_ICRS columns with units of degrees, and the positional uncertainty column with units of arcseconds. In the 'Output Rows' section, change the join type to be '1 not 2', then click 'Go' at the bottom to generate a new table containing 172 objects.

![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/1cf02ea3-5779-4d11-a411-3e15f647145f)

3.3) Remove anything within 2x its positional uncertainty of a GAIA EDR3 source. 

Repeat 3.1), instead using GAIA EDR3 as the remote table and the table generated in 3.2) as the local table. Repeat 3.2), instead setting table 1 and 2 to be the GAIA EDR3 crossmatch table just generated, with table 1 using the RA and DEC columns in degrees, and the positional uncertainty column in arcseconds, and table 2 using the ra_epoch2000 and dec_epoch2000 columns in degrees, and the positional uncertainty column in arcseconds. Generate the new table containing 89 objects.

![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/9f9a251d-f00f-456c-8a2e-a0411512f5c9)
![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/e0834a3d-9c3e-4b73-bf8a-58b35bdea247)

3.4) Remove anything within 5 arcseconds of a PANSTARRS DR1 source.

Repeat 3.1), instead using PANSTARRS DR1 as the remote table and the table generated in 3.3) as the local table. Using the matching rows button in the toolbar, select the 'Sky' algorithm with a maximum error of 5 arcseconds. Set table 1 and 2 to be the crossmatch table just generated, with table 1 using the RA and DEC columns, and table 2 using the RAJ2000 and DEJ2000 columns. Set the join type to '1 not 2' and generate the new table containing 72 objects.

![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/21cd333b-d301-4697-b05b-b2d763f0f20d)
![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/886cce81-71cc-400c-90be-853825193740)

3.5) Accessing the TYCHO-2 survey.

Select the VO drop-down menu at the top of the TOPCAT page, then click the 'VizieR Catalogue Service'. In 'Row Selection', ensure that 'All Rows' is selected, and that the maximum row count in unlimited. In 'Catalogue Selection', search by keyword for TYCHO-2, and select the one with name I/259 'The TYCHO-2 Catalogue'. Then click 'OK' at the bottom of the page. Be aware that this is a large catalogue and will take several minutes to upload onto TOPCAT.

![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/0afeb747-733f-448f-9126-afc81149be73)
![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/84b481c7-b91c-4a48-9f42-823d5ed76bc3)

3.6) Remove anything within 10 arcseconds of a TYCHO-2 source.

In matching rows, select the 'Sky' algorithm with a maximum error of 10 arcseconds. Set table 1 to be the table generated in 3.4), using the RA and DEC columns, and set table 2 to be the 'I_259_tyc2' table imported from VizieR, using the _RAJ2000 and _DEJ2000 columns. Set the join type to be '1 not 2', and generate the new table containing 67 objects.

![image](https://github.com/SaCu2001/ROSAT-data-filtering/assets/148392974/8b2444f3-2f42-4c3a-bd20-f8b6d0048ae8)

3.7) Accessing the Veron-Cetty survey.

Repeat 3.5), instead searching by keyword for Veron-Cetty, selecting the one with the name VII/258 'Quasars and Active Galactice Nuclei (13th Ed.)(Veron+ 2010)'. This may take several minutes to upload onto TOPCAT.

3.8) Removing anything within 10 arcseconds of a Veron-Cetty source.

Repeat 3.6), instead selecting table 1 to be the one generated in 3.6), using the RA and DEC columns, and selecting table 2 to be the 'VII_258_vv10' table imported from VizieR, using the _RAJ2000 and DEJ2000 columns. Generate the new table cotaining 57 objects. 

This is the list of potential XDIN candidates for further study. Finally, check that the 3 known XDINS within the SDSS all-sky survey are remaining in the generated list of 57 objects by comparing the RA and DEC of these 57 condidates with that of the Magnificent 7 XDINS:

![image](https://github.com/SaCu2001/ROSAT-data-filters/assets/148392974/54ed6570-0f38-4c99-bdb8-fe2ba7b589cf)
