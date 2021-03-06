require 'rubygems'
require "bundler/setup"
require 'appraisal'
require 'rake'
require 'rake/testtask'
require 'coveralls/rake/task'
require 'rubygems/package_task'
begin
  require 'cucumber/rake/task'
rescue LoadError
  $stderr.puts "Please install cucumber: `gem install cucumber`"
  exit 1
end
require './lib/airbrake/version'

Coveralls::RakeTask.new

task :default => ["test:unit", "test:integration"]

namespace :test do
  Rake::TestTask.new(:unit) do |t|
    t.libs << 'lib'
    t.pattern = 'test/*_test.rb'
    t.verbose = true
  end

  desc "Integration tests for all versions of Rails."
  task :integration do
    system 'INTEGRATION=true rake appraisal:rails-3.2 integration_test'\
    '&& INTEGRATION=true rake appraisal:rails-3.1 integration_test'\
    '&& INTEGRATION=true rake appraisal:rails-3.0 integration_test'\
    '&& rake coveralls:push'\
    '&& INTEGRATION=true rake appraisal cucumber'
  end
end

Rake::TestTask.new(:integration_test) do |t|
  t.libs << 'lib'
  t.pattern = 'test/integration.rb'
  t.verbose = true
end

namespace :changeling do
  desc "Bumps the version by a minor or patch version, depending on what was passed in."
  task :bump, :part do |t, args|
    # Thanks, Jeweler!
    if Airbrake::VERSION  =~ /^(\d+)\.(\d+)\.(\d+)(?:\.(.*?))?$/
      major = $1.to_i
      minor = $2.to_i
      patch = $3.to_i
      build = $4
    else
      abort
    end

    case args[:part]
    when /minor/
      minor += 1
      patch = 0
    when /patch/
      patch += 1
    else
      abort
    end

    version = [major, minor, patch, build].compact.join('.')

    File.open(File.join("lib", "airbrake", "version.rb"), "w") do |f|
      f.write <<EOF
module Airbrake
  VERSION = "#{version}".freeze
end
EOF
    end
  end

  desc "Writes out the new CHANGELOG and prepares the release"
  task :change do
    load 'lib/airbrake/version.rb'
    file    = "CHANGELOG"
    old     = File.read(file)
    version = Airbrake::VERSION
    message = "Bumping to version #{version}"
    editor = ENV["EDITOR"] || "vim" # prefer vim if no env variable for editor is set

    File.open(file, "w") do |f|
      f.write <<EOF
Version #{version} - #{Time.now}
===============================================================================

#{`git log $(git tag | grep -v rc | sort --version-sort | tail -1)..HEAD | git shortlog`}
#{old}
EOF
    end

    exec ["#{editor} #{file}",
          "git commit -aqm '#{message}'",
          "git tag -a -m '#{message}' v#{version}",
          "echo '\n\n\033[32mMarked v#{version} /' `git show-ref -s refs/heads/master` 'for release. Run: rake changeling:push\033[0m\n\n'"].join(' && ')
  end

  desc "Bump by a minor version (1.2.3 => 1.3.0)"
  task :minor do |t|
    Rake::Task['changeling:bump'].invoke(t.name)
    Rake::Task['changeling:change'].invoke
  end

  desc "Bump by a patch version, (1.2.3 => 1.2.4)"
  task :patch do |t|
    Rake::Task['changeling:bump'].invoke(t.name)
    Rake::Task['changeling:change'].invoke
  end

  desc "Push the latest version and tags"
  task :push do |t|
    system("git push origin master")
    system("git push origin $(git tag | grep -v rc | tail -1)")
  end
end

begin
  require 'yard'
  YARD::Rake::YardocTask.new do |t|
    t.files   = ['lib/**/*.rb', 'TESTING.rdoc']
  end
rescue LoadError
end

GEM_ROOT = File.dirname(__FILE__).freeze

desc "Clean files generated by rake tasks"
task :clobber => [:clobber_rdoc, :clobber_package]

LOCAL_GEM_ROOT = File.join(GEM_ROOT, 'tmp', 'local_gems').freeze

# Helper method that's used to only include relevant features when using
# various gemfiles. We don't want to, for instance, test sinatra features when 
# using the rails gemfile and vice versa.
def cucumber_opts
  opts = "--tags ~@wip "

  opts << ENV["FEATURE"] and return if ENV["FEATURE"]

  case ENV["BUNDLE_GEMFILE"]
  when /rails/
    opts << "features/rails.feature features/rails_with_js_notifier.feature features/metal.feature features/user_informer.feature"
  when /rack/
    opts << "features/rack.feature"
  when /sinatra/
    opts << "features/sinatra.feature"
  when /rake/
    opts << "features/rake.feature"
  end
end

Cucumber::Rake::Task.new(:cucumber) do |t|
  t.fork = true
  t.cucumber_opts = cucumber_opts
end
