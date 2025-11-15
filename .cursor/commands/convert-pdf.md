<inputs>
  <input name="input_file" type="file" required>
</inputs>

<variables>
  <variable name="input_file_directory">directory of {input_file}</variable>
  <variable name="input_file_name">basename of {input_file}</variable>
  <variable name="output_name" style="kebab-case">{input_file_name}</variable>
  <variable name="output_directory">{input_file_directory}/{output_name}/</variable>
  <variable name="output_html_directory">{output_directory}/html</variable>
  <variable name="output_md_directory">{output_directory}/md</variable>
</variables>

<outputs>
  <output>
    <directory name="{output_directory}">
      <description>The directory containing the converted markdown file</description>
      <file name="{output_md_directory}/{output_name}.md">The converted markdown file</file>
    </directory>
  </output>
</outputs>

<instructions>
  <instruction>
    Convert the PDF file to a high fidelity html document and then convert the html document to a markdown file
    <bash>
        ```bash
        mkdir -p "{output_html_directory}"
        pdftohtml -c -noframes "{input_file}" "{output_html_directory}/{output_name}.html"
        cd "{output_md_directory}"
        pandoc --from=html --to=gfm -f html-native_divs-native_spans \
          --resource-path="../html" \
          "../html/{output_name}.html" \
          -o "{output_name}.md" \
          --extract-media="."
        mv {input_file} {output_directory}/{output_name}.pdf
        ```
    </bash>
  </instruction>
  <instruction>
    Assume that pdftohtml and pandoc are installed, but iff not then stop and ask the operator if they want to install them via homebrew.
  </instruction>
</instructions>