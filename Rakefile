abort "Please use Ruby 1.9 to build AlfJS!" if RUBY_VERSION !~ /^1\.9/

require "bundler/setup"
require "erb"
require "uglifier"

# for now, the SproutCore compiler will be used to compile Ember.js
require "sproutcore"

LICENSE = File.read("generators/license.js")

## Some Ember modules expect an exports object to exist. Mock it out.

module SproutCore
  module Compiler
    class Entry
      def body
        "\n(function(exports) {\n#{@raw_body}\n})({});\n"
      end
    end
  end
end

## HELPERS ##

def strip_require(file)
  result = File.read(file)
  result.gsub!(%r{^\s*require\(['"]([^'"])*['"]\);?\s*}, "")
  result
end

def strip_ember_assert(file)
  result = File.read(file)
  result.gsub!(%r{^(\s)+ember_assert\((.*)\).*$}, "")
  result
end

def uglify(file)
  uglified = Uglifier.compile(File.read(file))
  "#{LICENSE}\n#{uglified}"
end

# Set up the intermediate and output directories for the interim build process

SproutCore::Compiler.intermediate = "tmp/intermediate"
SproutCore::Compiler.output       = "tmp/static"

# Create a compile task for an AlfJS package. This task will compute
# dependencies and output a single JS file for a package.
def compile_package_task(input, output=input)
  js_tasks = SproutCore::Compiler::Preprocessors::JavaScriptTask.with_input "packages/#{input}/lib/**/*.js", "."
  SproutCore::Compiler::CombineTask.with_tasks js_tasks, "#{SproutCore::Compiler.intermediate}/#{output}"
end

## TASKS ##

# Create alfjs:package tasks for each of the Ember packages
# eg. %w(package1 package2 package3)
namespace :alfjs do
  %w(vendor core utils).each do |package|
    task package => compile_package_task("alfjs-#{package}", "alfjs-#{package}")
  end
end

# Create a build task that depends on all of the package dependencies
task :build => ["alfjs:vendor", "alfjs:core", "alfjs:utils"]

distributions = {
  "alfresco" => ["alfjs-vendor", "alfjs-core", "alfjs-utils"]
}

distributions.each do |name, libraries|
  # Strip out require lines. For the interim, requires are
  # precomputed by the compiler so they are no longer necessary at runtime.
  file "dist/web/#{name}.js" => :build do
    puts "Generating #{name}.js"

    mkdir_p "dist/web"

    File.open("dist/web/#{name}.js", "w") do |file|
      libraries.each do |library|
        file.puts strip_require("tmp/static/#{library}.js")
      end
    end
  end

  # Minified distribution
  file "dist/web/#{name}.min.js" => "dist/web/#{name}.js" do
    require 'zlib'

    print "Generating #{name}.min.js... "
    STDOUT.flush

    File.open("dist/web/#{name}.prod.js", "w") do |file|
      file.puts strip_ember_assert("dist/web/#{name}.js")
    end

    minified_code = uglify("dist/web/#{name}.prod.js")
    File.open("dist/web/#{name}.min.js", "w") do |file|
      file.puts minified_code
    end

    gzipped_kb = Zlib::Deflate.deflate(minified_code).bytes.count / 1024

    puts "#{gzipped_kb} KB gzipped"

    rm "dist/web/#{name}.prod.js"
  end

  # Node.js Distribution
  file "dist/web/#{name}.js" => :build do
    puts "Generating Node Distribution #{name}.js"

    mkdir_p "dist/alfjs-node"

    cp "dist/web/#{name}.js", "dist/alfjs-node/#{name}.js"

    cp "package.json", "dist/alfjs-node/package.json"
  end

end


desc "Build AlfJS"
task :dist => distributions.keys.map {|name| "dist/web/#{name}.min.js"}

desc "Clean build artifacts from previous builds"
task :clean do
  sh "rm -rf tmp && rm -rf dist"
end



### RELEASE TASKS ###

ALFJS_VERSION = File.read("VERSION").strip

namespace :release do

  def pretend?
    ENV['PRETEND']
  end

  namespace :framework do
    desc "Update repo"
    task :update do
      puts "Making sure repo is up to date..."
      system "git pull" unless pretend?
    end

    desc "Update Changelog"
    task :changelog do
      last_tag = `git describe --tags --abbrev=0`.strip
      puts "Getting Changes since #{last_tag}"

      cmd = "git log #{last_tag}..HEAD --format='* %s'"
      puts cmd

      changes = `#{cmd}`
      output = "*Ember #{ALFJS_VERSION} (#{Time.now.strftime("%B %d, %Y")})*\n\n#{changes}\n"

      unless pretend?
        File.open('CHANGELOG', 'r+') do |file|
          current = file.read
          file.pos = 0;
          file.puts output
          file.puts current
        end
      else
        puts output.split("\n").map!{|s| "    #{s}"}.join("\n")
      end
    end

    desc "bump the version to the one specified in the VERSION file"
    task :bump_version, :version do
      puts "Bumping to version: #{ALFJS_VERSION}"

      unless pretend?
        # Bump the version of each component package
        Dir["packages/ember*/package.json", "ember.json"].each do |package|
          contents = File.read(package)
          contents.gsub! %r{"version": .*$}, %{"version": "#{ALFJS_VERSION}",}
          contents.gsub! %r{"(ember-?\w*)": [^\n\{,]*(,?)$} do
            %{"#{$1}": "#{ALFJS_VERSION}"#{$2}}
          end

          File.open(package, "w") { |file| file.write contents }
        end
      end
    end

    desc "Commit framework version bump"
    task :commit do
      puts "Commiting Version Bump"
      unless pretend?
        sh "git reset"
        sh %{git add VERSION CHANGELOG packages/**/package.json}
        sh "git commit -m 'Version bump - #{ALFJS_VERSION}'"
      end
    end

    desc "Tag new version"
    task :tag do
      puts "Tagging v#{ALFJS_VERSION}"
      system "git tag v#{ALFJS_VERSION}" unless pretend?
    end

    desc "Push new commit to git"
    task :push do
      puts "Pushing Repo"
      unless pretend?
        print "Are you sure you want to push the ember.js repo to github? (y/N) "
        res = STDIN.gets.chomp
        if res == 'y'
          system "git push"
          system "git push --tags"
        else
          puts "Not Pushing"
        end
      end
    end

    desc "Prepare for a new release"
    task :prepare => [:update, :changelog, :bump_version]

    desc "Commit the new release"
    task :deploy => [:commit, :tag, :push]
  end
end

task :default => :dist
