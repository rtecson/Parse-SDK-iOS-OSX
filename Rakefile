#
# Copyright (c) 2015-present, Parse, LLC.
# All rights reserved.
#
# This source code is licensed under the BSD-style license found in the
# LICENSE file in the root directory of this source tree. An additional grant
# of patent rights can be found in the PATENTS file in the same directory.
#

require_relative 'Vendor/xctoolchain/Scripts/xctask/build_task'
require_relative 'Vendor/xctoolchain/Scripts/xctask/build_framework_task'

script_folder = File.expand_path(File.dirname(__FILE__))
build_folder = File.join(script_folder, 'build')
release_folder = File.join(build_folder, 'release')
bolts_build_folder = File.join(script_folder, 'Vendor', 'Bolts-ObjC', 'build')

module Constants
  require 'plist'

  script_folder = File.expand_path(File.dirname(__FILE__))

  PARSE_CONSTANTS_HEADER = File.join(script_folder, 'Parse', 'PFConstants.h')
  PLISTS = [
    File.join(script_folder, 'Parse', 'Resources', 'Framework.plist'),
    File.join(script_folder, 'Parse', 'Resources', 'FrameworkOSX.plist'),
    File.join(script_folder, 'ParseStarterProject', 'iOS', 'ParseStarterProject', 'Resources', 'Info.plist'),
    File.join(script_folder, 'ParseStarterProject', 'iOS', 'ParseStarterProject-Swift', 'Resources', 'Info.plist'),
    File.join(script_folder, 'ParseStarterProject', 'OSX', 'ParseOSXStarterProject', 'Resources', 'Info.plist'),
    File.join(script_folder, 'ParseStarterProject', 'OSX', 'ParseOSXStarterProject-Swift', 'Resources', 'Info.plist')
  ]

  def self.current_version
    constants_file = File.open(PARSE_CONSTANTS_HEADER, 'r').read
    matches = constants_file.match(/(.*PARSE_VERSION\s*@")(.*)(")/)
    matches[2] # Return the second match, which is the version itself
  end

  def self.update_version(version)
    constants_file = File.open(PARSE_CONSTANTS_HEADER, 'r+')
    constants = constants_file.read
    constants.gsub!(/(.*PARSE_VERSION\s*@")(.*)(")/, "\\1#{version}\\3")

    constants_file.seek(0)
    constants_file.write(constants)

    PLISTS.each do |plist|
      update_info_plist_version(plist, version)
    end
  end

  def self.update_info_plist_version(plist_path, version)
    info_plist = Plist.parse_xml(plist_path)
    info_plist['CFBundleShortVersionString'] = version
    info_plist['CFBundleVersion'] = version
    File.open(plist_path, 'w') { |f| f.write(info_plist.to_plist) }
  end
end

namespace :build do
  desc 'Build iOS framework.'
  task :ios do
    task = XCTask::BuildFrameworkTask.new do |t|
      t.directory = script_folder
      t.build_directory = build_folder
      t.framework_type = XCTask::FrameworkType::IOS
      t.framework_name = 'Parse.framework'

      t.workspace = 'Parse.xcworkspace'
      t.scheme = 'Parse-iOS'
      t.configuration = 'Release'
    end
    result = task.execute
    unless result
      puts 'Failed to build iOS Framework.'
      exit(1)
    end
  end

  desc 'Build watchOS framework.'
  task :watchos do
    task = XCTask::BuildFrameworkTask.new do |t|
      t.directory = script_folder
      t.build_directory = build_folder
      t.framework_type = XCTask::FrameworkType::WATCHOS
      t.framework_name = 'Parse.framework'

      t.workspace = 'Parse.xcworkspace'
      t.scheme = 'Parse-watchOS'
      t.configuration = 'Release'
    end
    result = task.execute
    unless result
      puts 'Failed to build watchOS Framework.'
      exit(1)
    end
  end

  desc 'Build OS X framework.'
  task :osx do
    task = XCTask::BuildFrameworkTask.new do |t|
      t.directory = script_folder
      t.build_directory = build_folder
      t.framework_type = XCTask::FrameworkType::OSX
      t.framework_name = 'Parse.framework'

      t.workspace = 'Parse.xcworkspace'
      t.scheme = 'Parse-OSX'
      t.configuration = 'Release'
    end
    result = task.execute
    unless result
      puts 'Failed to build OS X Framework.'
      exit(1)
    end
  end
end

namespace :package do
  package_ios_name = 'Parse-iOS.zip'
  package_osx_name = 'Parse-OSX.zip'
  package_starter_ios_name = 'ParseStarterProject-iOS.zip'
  package_starter_osx_name = 'ParseStarterProject-OSX.zip'

  task :prepare do
    `rm -rf #{build_folder} && mkdir -p #{build_folder}`
    `rm -rf #{bolts_build_folder} && mkdir -p #{bolts_build_folder}`
  end

  desc 'Build and package all frameworks for the release'
  task :frameworks, [:version] => :prepare do |_, args|
    version = args[:version] || Constants.current_version
    Constants.update_version(version)

    ## Build iOS Framework
    Rake::Task['build:ios'].invoke
    bolts_path = File.join(bolts_build_folder, 'ios', 'Bolts.framework')
    ios_framework_path = File.join(build_folder, 'Parse.framework')
    make_package(release_folder,
                 [ios_framework_path, bolts_path],
                 package_ios_name)

    ## Build OS X Framework
    Rake::Task['build:osx'].invoke
    bolts_path = File.join(bolts_build_folder, 'osx', 'Bolts.framework')
    osx_framework_path = File.join(build_folder, 'Parse.framework')
    make_package(release_folder,
                 [osx_framework_path, bolts_path],
                 package_osx_name)
  end

  desc 'Build and package all starter projects for the release'
  task :starters, [:version] => :frameworks do |_, _args|
    require 'xcodeproj'

    ios_starters = [
      File.join(script_folder, 'ParseStarterProject', 'iOS', 'ParseStarterProject'),
      File.join(script_folder, 'ParseStarterProject', 'iOS', 'ParseStarterProject-Swift')
    ]
    ios_framework_archive = File.join(release_folder, package_ios_name)
    make_starter_package(release_folder, ios_starters, ios_framework_archive, package_starter_ios_name)

    osx_starters = [
      File.join(script_folder, 'ParseStarterProject', 'OSX', 'ParseOSXStarterProject'),
      File.join(script_folder, 'ParseStarterProject', 'OSX', 'ParseOSXStarterProject-Swift')
    ]
    osx_framework_archive = File.join(release_folder, package_osx_name)
    make_starter_package(release_folder, osx_starters, osx_framework_archive, package_starter_osx_name)
  end

  def make_package(target_path, items, archive_name)
    temp_folder = File.join(target_path, 'tmp')
    `mkdir -p #{temp_folder}`

    item_list = ''
    items.each do |item|
      `cp -R #{item} #{temp_folder}`

      file_name = File.basename(item)
      item_list << " #{file_name}"
    end

    archive_path = File.join(target_path, archive_name)
    `cd #{temp_folder}; zip -r --symlinks #{archive_path} #{item_list}`
    `rm -rf #{temp_folder}`
    puts "Release archive created: #{File.join(target_path, archive_name)}"
  end

  def make_starter_package(target_path, starter_projects, framework_archive, archive_name)
    starter_projects.each do |project_path|
      `git clean -xfd #{project_path}`
      `cd #{project_path} && unzip -o #{framework_archive}`

      xcodeproj_path = Dir.glob(File.join(project_path, '*.xcodeproj'))[0]
      prepare_xcodeproj(xcodeproj_path)
    end
    make_package(target_path, starter_projects, archive_name)

    starter_projects.each do |project_path|
      `git clean -xfd #{project_path}`
      `git checkout #{project_path}`
    end
  end

  def prepare_xcodeproj(path)
    project = Xcodeproj::Project.open(path)
    project.targets.each do |target|
      if target.name == 'Bootstrap'
        target.remove_from_project
      else
        target.dependencies.each(&:remove_from_project)
      end
    end
    project.save

    `rm -rf #{File.join(path, 'xcshareddata', 'xcschemes', '*')}`
  end
end

namespace :test do
  desc 'Run iOS Tests'
  task :ios do |_, args|
    task = XCTask::BuildTask.new do |t|
      t.directory = script_folder
      t.workspace = 'Parse.xcworkspace'

      t.scheme = 'Parse-iOS'
      t.sdk = 'iphonesimulator'
      t.destinations = ["\"platform=iOS Simulator,OS=9.0,name=iPhone 4s\"",
                        "\"platform=iOS Simulator,OS=9.0,name=iPhone 6 Plus\"",]
      t.configuration = 'Debug'
      t.additional_options = { "GCC_INSTRUMENT_PROGRAM_FLOW_ARCS" => "YES",
                               "GCC_GENERATE_TEST_COVERAGE_FILES" => "YES" }

      t.actions = [XCTask::BuildAction::TEST]
      t.formatter = XCTask::BuildFormatter::XCPRETTY
    end
    unless task.execute
      puts 'iOS Tests Failed!'
      exit(1)
    end
  end

  desc 'Run OS X Tests'
  task :osx do |_, args|
    task = XCTask::BuildTask.new do |t|
      t.directory = script_folder
      t.workspace = 'Parse.xcworkspace'

      t.scheme = 'Parse-OSX'
      t.sdk = 'macosx'
      t.destinations = ['arch=x86_64']
      t.configuration = 'Debug'
      t.additional_options = { "GCC_INSTRUMENT_PROGRAM_FLOW_ARCS" => "YES",
                               "GCC_GENERATE_TEST_COVERAGE_FILES" => "YES" }

      t.actions = [XCTask::BuildAction::TEST]
      t.formatter = XCTask::BuildFormatter::XCPRETTY
    end
    unless task.execute
      puts 'OS X Tests Failed!'
      exit(1)
    end
  end

  desc 'Run Deployment Tests'
  task :deployment do |_|
    Rake::Task['build:watchos'].invoke
    Rake::Task['package:frameworks'].invoke
    Rake::Task['package:starters'].invoke
  end

  desc 'Run Starter Project Tests'
  task :starters do |_|
    results = []
    ios_schemes = ['ParseStarterProject',
                   'ParseStarterProject-Swift']
    osx_schemes = ['ParseOSXStarterProject',
                   'ParseOSXStarterProject-Swift']

    ios_schemes.each do |scheme|
      task = XCTask::BuildTask.new do |t|
        t.directory = script_folder
        t.workspace = 'Parse.xcworkspace'

        t.scheme = scheme
        t.configuration = 'Debug'
        t.sdk = 'iphonesimulator'

        t.actions = [XCTask::BuildAction::CLEAN, XCTask::BuildAction::BUILD]
        t.formatter = XCTask::BuildFormatter::XCPRETTY
      end
      results << task.execute
    end
    osx_schemes.each do |scheme|
      task = XCTask::BuildTask.new do |t|
        t.directory = script_folder
        t.workspace = 'Parse.xcworkspace'

        t.scheme = scheme
        t.configuration = 'Debug'
        t.sdk = 'macosx'

        t.actions = [XCTask::BuildAction::CLEAN, XCTask::BuildAction::BUILD]
        t.formatter = XCTask::BuildFormatter::XCPRETTY
      end
      results << task.execute
    end

    results.each do |result|
      unless result
        puts 'Starter Project Tests Failed!'
        exit(1)
      end
    end
  end

  desc 'Run Podspec Lint'
  task :podspecs do |_|
    podspecs = ['Parse.podspec']
    results = []
    podspecs.each do |podspec|
      results << system("pod repo update; pod lib lint #{podspec} --verbose")
    end
    results.each do |result|
      unless result
        puts 'Podspec Tests Failed!'
        exit(1)
      end
    end
  end
end
