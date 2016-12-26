require 'ox'

TESTING_REPORT = "target/test-reports/TESTS-TestSuites.xml"
COVERAGE_REPORT = "target/test-reports/cobertura/coverage.xml"
CODENARC_REPORT = "target/CodeNarcReport.xml"

TESTING_REPORT_ARTIFACT = {:message => "Test Report", :path => "test-reports/index.html"}
COVERAGE_REPORT_ARTIFACT = {:message => "Coverage Report", :path => "cobertura/index.html"}
CODENARC_REPORT_ARTIFACT = {:message => "Codenarc Report", :path => "codenarc/CodeNarcReport.html"}

artifacts = []

if File.file?(TESTING_REPORT)
    junit.parse TESTING_REPORT
    junit.report
    
    artifacts << TESTING_REPORT_ARTIFACT
else
    warn "You are not running tests."
end

if File.file?(COVERAGE_REPORT)
    xml_string = File.read(COVERAGE_REPORT)
    @doc = Ox.parse(xml_string)
    
    message "Coverage: #{(@doc['line-rate'].to_f*100).to_i}%"
    
    cobertura = "# Coverage \n\n"
    
    attributes = ["Package", "Coverage"]
    
    cobertura << attributes.join(' | ') + "|\n"
    cobertura << attributes.map { |_| '---' }.join(' | ') + "|\n"
    
    @doc.nodes.last.nodes.each do |test|
        cobertura << "#{test['name']} | #{(test['line-rate'].to_f*100).to_i}%|\n"
    end
    
    markdown cobertura
    
    artifacts << COVERAGE_REPORT_ARTIFACT
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
    
    artifacts << CODENARC_REPORT_ARTIFACT
else
    warn "Codenarc report is missing."
end

# Based on https://github.com/samdmarshall/danger/blob/master/Dangerfile
if !artifacts.empty?
    username = ENV['CIRCLE_PROJECT_USERNAME']
    project_name = ENV['CIRCLE_PROJECT_REPONAME']
    build_number = ENV['CIRCLE_BUILD_NUM']
    node_index = ENV['CIRCLE_NODE_INDEX']
    should_display_message = username && project_name && build_number && node_index
    
    if should_display_message

        # build the path to where the circle CI artifacts will be uploaded to
        circle_ci_artifact_path  = 'https://circleci.com/api/v1/project/'
        circle_ci_artifact_path += username
        circle_ci_artifact_path += '/'
        circle_ci_artifact_path += project_name
        circle_ci_artifact_path += '/'
        circle_ci_artifact_path += build_number
        circle_ci_artifact_path += '/artifacts/'
        circle_ci_artifact_path += node_index
        circle_ci_artifact_path += '/$CIRCLE_ARTIFACTS/'
        
        artifacts.each do |artifact|
            # create a markdown link that uses the message text and the artifact path
            message_string = '[' + artifact[:message] + ']'
            message_string += '(' + circle_ci_artifact_path + artifact[:path] + ')'
            message(message_string)
        end
    end
end

# From https://github.com/Moya/Aeryn/blob/master/Dangerfile
has_test_changes = !git.modified_files.grep(/test/).empty?
warn("You didn't write new tests. Are you refactoring?") unless has_test_changes

# Warn summary on pull request
if github.pr_body.length < 5
    warn "Please provide a summary in the Pull Request description"
end

# From https://github.com/loadsmart/dangerfile/blob/master/Dangerfile
message("Good job on cleaning the code") if git.deletions > git.insertions
