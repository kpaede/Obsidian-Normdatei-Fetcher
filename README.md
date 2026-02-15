# Obsidian-Normdatei-Fetcher
Fetches from the GND (Gemeinsame Normdatei) into Obsidian. This repository contains a specialized Templater script for Obsidian that allows you to import data directly from the Gemeinsame Normdatei (GND) via the lobid.org API.

# Features
Live Search: Search for persons, subject headings, or corporate bodies directly within an Obsidian modal.

Preview Modal: View record details before importing and toggle specific fields on or off.

Field Mapping: Automatically translate technical GND keys (e.g., dateOfBirth) into your preferred Obsidian property names (e.g., born) via a mapping file.

Image Integration: Automatically detects the depiction field and embeds the image URL as a Markdown image under the main heading.

Biographical Data: Imports historical/biographical information and generates links to the DNB Catalog and GND Explorer.

Installation and Setup
1. Prerequisites
Install the Templater community plugin.

2. Create the Mapping File
The script looks for a file named field-mapping.md in your vault to determine how to rename incoming data fields. Create this file and add your mappings in Key: Value format: or take my field as a start.

Example content for the Mapping file

preferredName: name
dateOfBirth: born
dateOfDeath: died
placeOfBirth: location
biographicalOrHistoricalInformation: biography
depiction: portrait

3. Setup the Template
Create a new file in your Obsidian templates folder (e.g., GND-Import.md).

Paste the provided JavaScript Code (or take my file from this Repository).

Ensure the script is wrapped correctly in Templater's execution tags: <%* ... %>.

# Tutorial
Open a Note: Navigate to the note where you want to import the data and place the cursor in the note.

Run Template: Trigger the Templater "Insert Template" command.

Search: Type your search query into the input field.

Select: Click on the desired entry from the results list.

Review: In the preview window, check the boxes for the fields you wish to import. Fields defined in your mapping file are selected by default.

Import: Click the "Import" button.

Result
The script will:

Update the note's Frontmatter (YAML) with the selected fields.

Insert a Level 1 Heading with the preferred name.

Embed the portrait image (if available and selected) directly under the heading.

Append the biography and a list of external GND links.

# Customization
CSS Styling
The visual appearance of the modal (colors, width, and badges) can be adjusted within the // --- 2. CSS FOR THE UI --- section of the script.

# Data Harvesting
The script uses a harvest() function to process complex JSON objects from the API. If you need to extract specific sub-fields from the GND data, you can modify this function to handle different object structures.

License and Data Source
Data is fetched via lobid.org. Please refer to the lobid GND API documentation for terms of use and data licensing information.
