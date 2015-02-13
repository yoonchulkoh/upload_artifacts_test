=begin
This script required latest betabuilder gem.
Please follow below commands.

git clone https://github.com/lukeredpath/betabuilder.git
cd betabuilder.git
sed -i -e "s/0\.7\.4\.1/0.7.4.1.1/" betabuilder.gemspec
gem build betabuilder.gemspec
sudo gem install ./betabuilder-0.7.4.1.1.gem
=end
Encoding.default_external = Encoding::UTF_8
require "rubygems/version"
require "rake/clean"
require "date"
require 'time'
require "json"
require "open3"

# Application info
APP_NAME = "upload_artifacts_test"
INFO_PLIST = File.expand_path("#{APP_NAME}/#{APP_NAME}/Info.plist")



# Code signing
CODE_SIGN_IDENTITY = "iPhone Developer"

# Build paths
WORKSPACE = File.expand_path("#{APP_NAME}/#{APP_NAME}.xcodeproj")
BUILD_DIR = File.expand_path("build")
TEMP_DIR = File.expand_path("#{BUILD_DIR}/temp")
ARCHIVE_PATH = File.expand_path("#{BUILD_DIR}/#{APP_NAME}.xcarchive")
ARCHIVE_ZIP_FILE = File.expand_path("#{BUILD_DIR}/#{APP_NAME}.xcarchive.zip")
IPA_FILE = File.expand_path("#{BUILD_DIR}/#{APP_NAME}.ipa")
DSYM_FILE = File.expand_path("#{BUILD_DIR}/#{APP_NAME}.app.dSYM")
DSYM_ZIP_FILE = File.expand_path("#{BUILD_DIR}/#{APP_NAME}.app.dSYM.zip")

CLEAN.include(BUILD_DIR)
CLOBBER.include(BUILD_DIR)

desc 'upload_artifacts'
task :upload do
  upload_artifacts
end

task :export_ipa do
  clean_ipa
  export_ipa
end

def clean_ipa
  system "rm -f #{IPA_FILE}"
end

def export_ipa
  sh 'xcodebuild',
    '-exportArchive',
    '-exportFormat', 'IPA',
    '-archivePath', "#{BUILD_DIR}/#{APP_NAME}.xcarchive",
    '-exportPath', "#{BUILD_DIR}/#{APP_NAME}.ipa",
    '-exportProvisioningProfile', PROFILE_NAME_ADHOC
end

desc 'Archive release build and deploy to hockey.'
task :release do
  Rake::Task["clean"].invoke

  changelog = `curl -s "#{ENV['JOB_URL']}#{ENV['BUILD_NUMBER']}/api/xml?wrapper=changes&xpath=//changeSet//comment"`.scan(%r{<comment>(.*?)</comment>}m).join("\n")
  changelog = 'new build' if changelog.empty?

  Dir.chdir "#{APP_NAME}"
  # sh 'pod install'
  # Dir.chdir '../'

  # xcarchive作成
  sh 'xcodebuild',
    '-sdk', 'iphoneos',
    # '-workspace', WORKSPACE,
    '-scheme', "#{APP_NAME}",
    '-configuration', 'Release',
    "CONFIGURATION_BUILD_DIR=#{BUILD_DIR}",
    "CONFIGURATION_TEMP_DIR=#{TEMP_DIR}",
    "CODE_SIGN_IDENTITY=#{CODE_SIGN_IDENTITY}",
    'archive',
    '-archivePath', ARCHIVE_PATH

  clean_ipa
  # export_ipa

  # zip化
  sh %[(cd #{BUILD_DIR}; zip -ryq #{APP_NAME}.app.dSYM.zip #{APP_NAME}.app.dSYM)]
  sh %[mv #{DSYM_FILE} #{BUILD_DIR}/#{APP_NAME}.xcarchive/dSYMs/#{APP_NAME}.app.dSYM]
  sh %[(cd #{BUILD_DIR}; zip -ryq #{APP_NAME}.xcarchive.zip #{APP_NAME}.xcarchive)]
end

def upload_artifacts
  owner = "yoonchulkoh"
  repo = "upload_artifacts_test"
  version = "#{InfoPlist.marketing_version}-#{InfoPlist.build_version}"
  tag_name = "v#{version}_#{Time.now.utc.iso8601.gsub(':', '-')}"
  branch = "master"
  access_token = ENV["GITHUB_ACCESS_TOKEN"]
 
  upload_date = DateTime.now.strftime("%Y/%m/%d %H:%M:%S")
  build_version = InfoPlist.marketing_and_build_version
 
  changelog = "Build: #{build_version}\nUploaded: #{upload_date}\n"
  changelog << %x[git log --date=short --pretty=format:"- [%h](http://github.com/#{owner}/#{repo}/commit/%H) - %s _%cd_" --no-merges $(git describe --abbrev=0 --tags)..]
 
  release_data = {tag_name: tag_name, target_commitish: branch, name: tag_name, body: changelog, draft: false, prerelease: false}.to_json
  create_release_url = %[https://api.github.com/repos/#{owner}/#{repo}/releases?access_token=#{access_token}]
 
  curl_command = ["curl", "-sSf", "-d", "#{release_data}", "#{create_release_url}"]
  out, status = Open3.capture2(*(curl_command))
 
  release_id = JSON.parse(out)["id"]
  upload_url = %[https://uploads.github.com/repos/#{owner}/#{repo}/releases/#{release_id}/assets?name=#{APP_NAME}-#{version}.xcarchive.zip]
  sh %[curl -sSf -w "%{http_code} %{url_effective}\\n" -o /dev/null -X POST #{upload_url} -H "Accept: application/vnd.github.v3+json" -H "Authorization: token #{access_token}" -H "Content-Type: application/zip" --data-binary @"#{ARCHIVE_ZIP_FILE}"]
end


namespace :version do
  module InfoPlist
    extend self
 
    def [](key)
      output = %x[/usr/libexec/PlistBuddy -c "Print #{key}" #{INFO_PLIST}].strip
      raise "The key `#{key}' does not exist in `#{INFO_PLIST}'." if output.include?('Does Not Exist')
      output
    end
 
    def set(key, value, file = "#{INFO_PLIST}")
      %x[/usr/libexec/PlistBuddy -c 'Set :#{key} "#{value}"' '#{file}'].strip
    end
    def []=(key, value)
      set(key, value)
    end
 
    def build_version
      self['CFBundleVersion']
    end
    def build_version=(revision)
      self['CFBundleVersion'] = revision
    end
 
    def marketing_version
      self['CFBundleShortVersionString']
    end
    def marketing_version=(version)
      self['CFBundleShortVersionString'] = version
    end
 
    def bump_marketing_version_segment(segment_index)
      segments = Gem::Version.new(marketing_version).segments
      segments[segment_index] = segments[segment_index].to_i + 1
      (segment_index+1..segments.size-1).each { |i| segments[i] = 0 }
      version = segments.map(&:to_i).join('.')
      puts "Setting marketing version to: #{version}"
      self.marketing_version = version
      system("git commit #{INFO_PLIST} -m 'Bump to #{marketing_and_build_version}'")
    end
 
    def marketing_and_build_version
      "#{marketing_version} (#{build_version})"
    end
 
    def update_build_number
      build_number = %x[git rev-list HEAD | wc -l | tr -d " "].strip.to_i
      self.build_version = (build_number+1).to_s
    end
 
    def update_build_version
      rev = ""
      pr_number = ENV["TRAVIS_PULL_REQUEST"]
      if pull_request?
        rev << %x[git rev-parse --short HEAD^2].strip
        rev << " ##{pr_number}"
      else
        rev << %x[git rev-parse --short HEAD].strip
      end
 
      puts "Setting build version to: #{rev}"
 
      InfoPlist.build_version = rev
    end
  end
 
  desc "Print the current version"
  task :current do
    puts InfoPlist.marketing_and_build_version
  end
 
  desc "Sets build version to last git commit (happens on each build)"
  task :update_build_version do
    InfoPlist.update_build_version
  end
 
  desc "Sets build number to last git commit count (happens on each build)"
  task :update_build_number do
    InfoPlist.update_build_number
  end
 
  namespace :bump do
    desc "Bump patch version (0.0.X)"
    task :patch do
      InfoPlist.update_build_number
      InfoPlist.bump_marketing_version_segment(2)
    end
 
    desc "Bump minor version (0.X.0)"
    task :minor do
      InfoPlist.update_build_number
      InfoPlist.bump_marketing_version_segment(1)
    end
 
    desc "Bump major version (X.0.0)"
    task :major do
      InfoPlist.update_build_number
      InfoPlist.bump_marketing_version_segment(0)
    end
  end
end