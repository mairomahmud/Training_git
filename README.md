#!/usr/bin/env python3
import os
import xml.etree.ElementTree as ET
import csv
import pandas as pd

# Base directory containing plasmidfinder output folders
base_dir = os.path.expanduser("~/training_folder/plasmid_output")
output_tsv = os.path.join(base_dir, "all_plasmidfinder_results.tsv")
output_excel = os.path.join(base_dir, "all_plasmidfinder_combinedresults.xlsx")

# TSV header
header = ["Sample", "File", "Plasmid", "Identity (%)", "Coverage (%)", "Accession"]

rows = []

print("🔍 Scanning for XML files with plasmid hits...")

# Loop through each sample folder
for sample_dir in os.listdir(base_dir):
    sample_path = os.path.join(base_dir, sample_dir)
    tmp_path = os.path.join(sample_path, "tmp")

    if os.path.isdir(tmp_path):
        for xml_file in os.listdir(tmp_path):
            if xml_file.endswith(".xml"):
                xml_path = os.path.join(tmp_path, xml_file)
                try:
                    tree = ET.parse(xml_path)
                    root = tree.getroot()

                    # Extract query length
                    query_len = root.findtext(".//BlastOutput_query-len", default="1")
                    query_len = int(query_len)

                    # Loop through all Hit elements in Iteration_hits
                    for hit in root.findall(".//Iteration_hits/Hit"):
                        plasmid = hit.findtext("Hit_def", default="NA")
                        accession = hit.findtext("Hit_accession", default="NA")

                        # Extract HSP details (score/identity section)
                        hsp = hit.find(".//Hsp")
                        if hsp is not None:
                            identity = hsp.findtext("Hsp_identity", default="0")
                            align_len = hsp.findtext("Hsp_align-len", default="1")

                            identity_pct = round((int(identity) / int(align_len)) * 100, 2)
                            coverage_pct = round((int(align_len) / query_len) * 100, 2)

                            rows.append([sample_dir, xml_file, plasmid, identity_pct, coverage_pct, accession])

                except ET.ParseError:
                    print(f"⚠️ Could not parse XML: {xml_file}")
                except Exception as e:
                    print(f"⚠️ Error processing {xml_file}: {e}")

# Write to TSV
with open(output_tsv, "w", newline="") as f:
    writer = csv.writer(f, delimiter="\t")
    writer.writerow(header)
    writer.writerows(rows)

# Export to Excel using pandas
df = pd.DataFrame(rows, columns=header)
df.to_excel(output_excel, index=False)

print(f"✅ Combined TSV saved to: {output_tsv}")
print(f"✅ Excel file saved to: {output_excel}")
print("🎉 Done! Check your files.")
