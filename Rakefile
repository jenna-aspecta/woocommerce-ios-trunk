# frozen_string_literal: true

require 'fileutils'
require 'tmpdir'
require 'rake/clean'
require 'yaml'
require 'digest'

PROJECT_DIR = __dir__
SWIFTLINT_BIN = File.join(PROJECT_DIR, 'Pods', 'SwiftLint', 'swiftlint')
XCODE_WORKSPACE = 'WooCommerce.xcworkspace'
XCODE_SCHEME = 'WooCommerce'
XCODE_CONFIGURATION = 'Debug'

task default: %w[test]

desc 'Install required dependencies'
task dependencies: %w[dependencies:check]

namespace :dependencies do
  task check: %w[bundler:check bundle:check credentials:apply pod:check lint:check]

  namespace :bundler do
    task :check do
      Rake::Task['dependencies:bundler:install'].invoke unless command?('bundler')
    end

    task :install do
      puts 'Bundler not found in PATH, installing to vendor'
      ENV['GEM_HOME'] = File.join(PROJECT_DIR, 'vendor', 'gems')
      ENV['PATH'] = File.join(PROJECT_DIR, 'vendor', 'gems', 'bin') + ":#{ENV.fetch('PATH', nil)}"
      sh 'gem install bundler' unless command?('bundler')
    end
    CLOBBER << 'vendor/gems'
  end

  namespace :bundle do
    task :check do
      sh 'bundle check --path=${BUNDLE_PATH:-vendor/bundle} > /dev/null', verbose: false do |ok, _res|
        next if ok

        # bundle check exits with a non zero code if install is needed
        dependency_failed('Bundler')
        Rake::Task['dependencies:bundle:install'].invoke
      end
    end

    task :install do
      fold('install.bundler') do
        sh 'bundle install --jobs=3 --retry=3 --path=${BUNDLE_PATH:-vendor/bundle}'
      end
    end
    CLOBBER << 'vendor/bundle'
    CLOBBER << '.bundle'
  end

  namespace :credentials do
    task :apply do
      next unless Dir.exist?(File.join(Dir.home, '.mobile-secrets/.git')) || ENV.key?('CONFIGURE_ENCRYPTION_KEY')

      sh('FASTLANE_SKIP_UPDATE_CHECK=1 FASTLANE_ENV_PRINTER=1 bundle exec fastlane run configure_apply force:true')
    end
  end

  namespace :pod do
    task :check do
      unless podfile_locked? && lockfiles_match?
        dependency_failed('CocoaPods')
        Rake::Task['dependencies:pod:install'].invoke
      end
    end

    task :install do
      fold('install.cocoapds') do
        pod %w[install --repo-update]
      end
    end

    task :clean do
      fold('clean.cocoapds') do
        FileUtils.rm_rf('Pods')
      end
    end
    CLOBBER << 'Pods'
  end

  namespace :lint do
    task :check do
      if swiftlint_needs_install
        dependency_failed('SwiftLint')
        Rake::Task['dependencies:pod:install'].invoke
      end
    end
  end
end

CLOBBER << 'vendor'

desc 'Mocks'
task :mocks do
  sh './WooCommerce/WooCommerceUITests/Mocks/scripts/start.sh'
end

desc "Build #{XCODE_SCHEME}"
task build: [:dependencies] do
  xcodebuild(:build)
end

desc "Profile build #{XCODE_SCHEME}"
task buildprofile: [:dependencies] do
  ENV['verbose'] = '1'
  xcodebuild(:build,
             "OTHER_SWIFT_FLAGS='-Xfrontend -debug-time-compilation -Xfrontend -debug-time-expression-type-checking'")
end

task timed_build: [:clean] do
  require 'benchmark'
  time = Benchmark.measure do
    Rake::Task['build'].invoke
  end
  puts "CPU Time: #{time.total}"
  puts "Wall Time: #{time.real}"
end

desc 'Run test suite'
task test: [:dependencies] do
  xcodebuild(:build, :test)
end

desc 'Remove any temporary products'
task :clean do
  xcodebuild(:clean)
end

desc 'Checks the source for style errors'
task lint: %w[dependencies:lint:check] do
  swiftlint %w[lint --quiet]
end

namespace :lint do
  desc 'Automatically corrects style errors where possible'
  task autocorrect: %w[dependencies:lint:check] do
    swiftlint %w[lint --autocorrect --quiet]
  end
end

desc 'Open the project in Xcode'
task xcode: [:dependencies] do
  sh "open #{XCODE_WORKSPACE}"
end

desc 'Run all code generation tasks'
task :generate do
  %w[Hardware Networking Storage Yosemite WooCommerce WooFoundation].each do |prefix|
    puts "\n\nGenerating Copiable for #{prefix}..."
    puts '=' * 100

    sh "./Pods/Sourcery/bin/sourcery --config CodeGeneration/Sourcery/Copiable/#{prefix}-Copiable.sourcery.yaml"
  end

  puts "\n\nDONE. Generated Copiable for all projects."

  %w[Hardware Networking Yosemite WooFoundation].each do |prefix|
    puts "\n\nGenerating Fakes for #{prefix}..."
    puts '=' * 100

    sh "./Pods/Sourcery/bin/sourcery --config CodeGeneration/Sourcery/Fakes/#{prefix}-Fakes.yaml"
  end

  puts "\n\nDONE. Generated Fakes."
end

def fold(label)
  puts "travis_fold:start:#{label}" if travis?
  yield
  puts "travis_fold:end:#{label}" if travis?
end

def travis?
  !ENV['TRAVIS'].nil?
end

def pod(args)
  args = %w[bundle exec pod] + args
  sh(*args)
end

def lockfiles_match?
  File.file?('Pods/Manifest.lock') && FileUtils.compare_file('Podfile.lock', 'Pods/Manifest.lock')
end

def podfile_locked?
  podfile_checksum = Digest::SHA1.file('Podfile')
  lockfile_checksum = YAML.load_file('Podfile.lock')['PODFILE CHECKSUM']

  podfile_checksum == lockfile_checksum
end

def swiftlint_needs_install
  # Notice that this doesn't check whether the local version is up-to-date.
  # Given we are using CocoaPods to install SwiftLint, it's safe to assume SwiftLint will be up-to-date most of the time.
  # We are trading being 100% sure we're up-to-date for a faster check.
  File.exist?(SWIFTLINT_BIN) == false
end

def swiftlint(args)
  args = [SWIFTLINT_BIN] + args
  sh(*args)
end

def xcodebuild(*build_cmds)
  cmd = 'xcodebuild'
  cmd += " -destination 'platform=iOS Simulator,name=iPhone 6s'"
  cmd += ' -sdk iphonesimulator'
  cmd += " -workspace #{XCODE_WORKSPACE}"
  cmd += " -scheme #{XCODE_SCHEME}"
  cmd += " -configuration #{xcode_configuration}"
  cmd += ' '
  cmd += build_cmds.map(&:to_s).join(' ')
  cmd += ' | bundle exec xcpretty -f `bundle exec xcpretty-travis-formatter` && exit ${PIPESTATUS[0]}' unless ENV['verbose']
  sh(cmd)
end

def xcode_configuration
  ENV['XCODE_CONFIGURATION'] || XCODE_CONFIGURATION
end

def command?(command)
  system("which #{command} > /dev/null 2>&1")
end

def dependency_failed(component)
  msg = "#{component} dependencies missing or outdated. "
  if ENV['DRY_RUN']
    msg += 'Run rake dependencies to install them.'
    raise msg
  else
    msg += 'Installing...'
    puts msg
  end
end

def check_dependencies_hook
  ENV['DRY_RUN'] = '1'
  begin
    Rake::Task['dependencies'].invoke
  rescue StandardError => e
    puts e.message
    exit 1
  end
end