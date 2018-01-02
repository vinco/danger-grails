require 'ox'

CIRCLE_FILE_LIST = ["circle.yml", ".circleci/config.yml"]

TESTING_REPORT = "build/reports/test/xml/TESTS-TestSuites.xml"
COVERAGE_REPORT = "build/reports/clover/clover.xml"
CODENARC_REPORT = "build/reports/codenarc/codenarc.xml"

pr_to_master = github.branch_for_base == "master"
valid_title_for_master = (github.pr_title.include? "(hotfix)") || (github.pr_title.include? "-> master")
if pr_to_master && !valid_title_for_master
    github.api.close_pull_request(github.pr_json["base"]["repo"]["full_name"], github.pr_json["number"])
    fail "Invalid PR to master! Only integration or hotfix PR are allowed in master branch."
end

pr_to_core_repo = github.pr_json["base"]["repo"]["name"].include? "-core"
if pr_to_core_repo && !(git.commits.last =~ /version:/)
    fail "It is necessary to update core version"
end

if CIRCLE_FILE_LIST.any? { |file| git.modified_files.include?(file) }
  warn "You are updating circle."
end

if File.file?(TESTING_REPORT)
    junit.parse TESTING_REPORT
    junit.report
else
    warn "You are not running tests."
end

if File.file?(COVERAGE_REPORT)
    xml_string = File.read(COVERAGE_REPORT)
    doc = Ox.parse(xml_string)
    
    coveredelements = doc.nodes.first.nodes.first.nodes.first['coveredelements'].to_f
    elements = doc.nodes.first.nodes.first.nodes.first['elements'].to_f
    coverage = 100*coveredelements/elements
    
    message "Coverage: #{coverage.round(2)}%"
    
    cobertura = "# Coverage \n\n"
    
    attributes = ["Package", "Coverage"]
    
    cobertura << attributes.join(' | ') + "|\n"
    cobertura << attributes.map { |_| '---' }.join(' | ') + "|\n"
    
    doc.nodes.last.nodes.first.nodes[1..-1].each do |test|
        name = test['name']
        coveredelements = test.nodes.first['coveredelements'].to_f
        elements = test.nodes.first['elements'].to_f
        coverage = 100*coveredelements/elements

        cobertura << "#{name} | #{coverage.round(2)}%|\n"
    end
    
    markdown cobertura
else
    warn "Coverage report is missing."
end

if File.file?(CODENARC_REPORT)
    xml_string = File.read(CODENARC_REPORT)
    @doc = Ox.parse(xml_string)
    
    summary = @doc.nodes.first.locate("PackageSummary").first
    packages = @doc.nodes.first.locate("Package").select {|item| item.nodes.size > 0}
    
    files = summary['totalFiles']
    files_with_violations = summary['filesWithViolations']
    priority_1 = summary['priority1']
    priority_2 = summary['priority2']
    priority_3 = summary['priority3']
    
    codenarc_md = "# Static Program Analysis \n\n"
    
    attributes = ["Files", "Files with violations", "Priority 1", "Priority 2", "Priority 3"]
    codenarc_md << attributes.join(' | ') + "|\n"
    codenarc_md << attributes.map { |_| '---' }.join(' | ') + "|\n"
    codenarc_md << "#{files} | #{files_with_violations} | #{priority_1} | #{priority_2} | #{priority_3} |\n"
    
    if (files_with_violations.to_i > 0)
        attributes = ["Line", "Rule", "Priority", "Source", "Message"]
    
        packages.each do |package|
            package.nodes.each do |file|
                codenarc_md << "#### #{file['name']}\n"
                codenarc_md << attributes.join(' | ') + "|\n"
                codenarc_md << attributes.map { |_| '---' }.join(' | ') + "|\n"
                
                file.nodes.each do |violation|
                    codenarc_md << "#{violation['lineNumber']}|"
                    codenarc_md << "#{violation['ruleName']}|"
                    codenarc_md << "#{violation['priority']}|"
                    codenarc_md << "`#{violation.nodes.first.nodes.first.value}`|"
                    codenarc_md << "#{violation.nodes.last.nodes.first.value}|\n"
                end
                
                codenarc_md << "\n"
            end
        end
    end
    
    markdown codenarc_md
else
    warn "Codenarc report is missing."
end

# From https://github.com/Moya/Aeryn/blob/master/Dangerfile
has_test_changes = !(git.modified_files + git.added_files).grep(/test/).empty?
warn("You didn't write new tests. Are you refactoring?") unless has_test_changes

# Warn summary on pull request
if github.pr_body.length < 5
    warn "Please provide a summary in the Pull Request description"
end

# From https://github.com/loadsmart/dangerfile/blob/master/Dangerfile
message("Good job on cleaning the code") if git.deletions > git.insertions