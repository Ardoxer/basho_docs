### Basho Docs' Rakefile
#
# This Rakefile is to be invoked with the `rake` Ruby Gem. It's best if the that
# Gem is installed using Bundler and the included Gemfile.
#
# This file will act as the canonical builder for the JavaScript/CoffeeScript
# and CSS/SCSS 'dynamic' files, which are compiled into static/js/ and
# static/css/ respectively.

#TODO<drew.pirrone.brusse@gmail>: Make sure the above description is correct and
# complete.

require_relative 'rake_libs/compile_js'
require_relative 'rake_libs/s3_deploy'
require_relative 'rake_libs/downloads_metadata_generator'

$css_source = "./dynamic/css"
$css_dest   = "./static/css"
$js_source  = "./dynamic/js"
$js_dest    = "./static/js"
$cache_dir  = "./dynamic/.cache"
$hugo_dest  = "./public" # Should always be set to `publishdir` from config.yml

### Rake directory definitions
directory "#{$js_dest}"
directory "#{$css_dest}"
directory "#{$cache_dir}"

#TODO<drew.pirrone.brusse@gmail>: Check to make sure Hugo's version is >= 0.15


######################################################################
### Rake Namespace and Task definitions

##########
# Default
Rake::TaskManager.record_task_metadata = true
task :default do
  puts("Basho Documentation Generate System Usage:")
  puts("")
  Rake::application.options.show_tasks = :tasks  # this solves sidewaysmilk problem
  Rake::application.options.show_task_pattern = //
  Rake::application.display_tasks_and_comments
end;


########
# Clean
#TODO<drew.pirrone.brusse@gmail>: These `rm -rf`s are maybe a bit much? Should
# we be more precise with what we delete (and, if so, how)?
desc      "Clean dynamically generated content (does not clean Hugo content)"
task      :clean => ['clean:js', 'clean:css']
namespace :clean do
  desc "Clean dynamically generated JS"
  task :js do
    js_file_list = Dir["#{$js_dest}/**/**"]
    js_file_list.each do |f|
      log_dir_deletion(f)
      FileUtils.rm(f)
    end
  end
  desc "Clean dynamically generated CSS"
  task :css do
    css_file_list = Dir["#{$css_dest}/**/**"]
    css_file_list.each do |f|
      log_dir_deletion(f)
      FileUtils.rm(f)
    end
  end
  desc "Clean Hugo-generated content"
  task :hugo do
    log_dir_deletion($hugo_dest)
    FileUtils.rm_rf($hugo_dest)
  end
end


########
# Build
desc      "Compile compressed JS and compressed CSS"
task      :build => ['clean', 'build:js', 'build:css']
namespace :build do
  task :js  => ["#{$js_dest}", 'clean:js']   do compile_js(debug: false); end
  task :css => ["#{$css_dest}", 'clean:css'] do compile_css(debug: false); end

  ################
  # Build : Debug
  desc      "Compile human-readable JS and compile human-readable CSS"
  task      :debug => ['clean', 'build:debug:js', 'build:debug:css']
  namespace :debug do
    desc "Compile human-readable JS"
    task :js  => ["#{$js_dest}", 'clean:js']   do compile_js(debug: true); end

    desc "Compile human-readable CSS"
    task :css => ["#{$css_dest}", 'clean:css'] do compile_css(debug: true); end
  end
end


########
# Watch
desc      "Rebuild compressed JS and CSS content on file saves"
task      :watch do sh 'bundle exec guard -g css js'; end
namespace :watch do

  task :js  do sh 'bundle exec guard -g js'; end
  task :css do sh 'bundle exec guard -g css'; end

  ################
  # Watch : Debug
  desc      "Rebuild human-readable JS and CSS content on file saves"
  task      :debug => ['clean'] do sh 'bundle exec guard -g debug_js debug_css'; end
  namespace :debug do
    task :js  do sh 'bundle exec guard -g debug_js'; end
    task :css do sh 'bundle exec guard -g debug_css'; end
  end
end


#######
# Hugo
desc      "Generate the static site into #{$hugo_dest}"
task      :hugo => ['clean:hugo'] do sh "hugo -d #{$hugo_dest}"; end
namespace :hugo do

  #TODO<drew.pirrone.brusse@gmail>: Add in some way to specify ip/port.
  desc "Run Hugo Server"
  task :server do sh "hugo server"; end
end


#########
# Deploy
desc      "Build and deploy static artifacts"
task      :deploy => [
                      'clean',
                      'deploy:fetch_archived_content',
                      'build:js',
                      'build:css',
                      'hugo'
                     ] do do_deploy(); end
namespace :deploy do
  task :immediately_and_unsafely do do_deploy(); end
  task :fetch_archived_content do do_fetch_archived_content(); end
end

# Generate download.yaml metadata.
#   This task should be run every time a new package is placed onto the
#   downloads server, and the updated file(s) should be tracked in Git.
desc "Generate package URI information"
task :generate_downloads_metadata do generate_downloads_metadata(); end


######################################################################
### Helper/Compilation functions

# Helper function that will print "    deleting #{dir_name}" to the console, and
# color the "deleting" text red.
def log_dir_deletion(dir_name)
  red = "\033[31m"
  nc  = "\033[0m" # no color
  print "    #{red}deleting#{nc} #{dir_name}\n"
end

# Helper function that will print "    write #{file_name}" to the console, and
# color the "write" text green. This is to match the Compass log output on file
# writes.
def log_js_write(file_name)
  green = "\033[32m"
  nc    = "\033[0m" # no color
  print "    #{green}write#{nc} #{file_name}\n"
end

def compile_css(debug: false)
  # This code was largely inspired by this SO question
  # http://stackoverflow.com/questions/25399962/compass-from-ruby-sasscompiler-not-found
  # with addtional code and configuraion options drawn from:
  # https://github.com/Compass/compass/blob/350bcaa544f594bca972aaa29a9bdfddceee5d4f/cli/lib/compass/exec/global_options_parser.rb
  # https://github.com/Compass/compass/blob/350bcaa544f594bca972aaa29a9bdfddceee5d4f/cli/lib/compass/exec/project_options_parser.rb

  #NOTE<drew.pirrone.brusse@gmail>: I made the choice here to start from the
  # above SO question and code snippets to make this call using the undocumented
  # Ruby API of Compass rather than building a shell call out of a Ruby string
  # and passing that to the OS layer. I don't know if it's the right choice, but
  # it's the choice I made. Hopefully no one gets bit by it.
  require 'compass'
  require 'compass/sass_compiler'
  require 'sass-css-importer'

  Compass.configuration.add_import_path "#{$css_source}/imports"
  configs = {}
  configs[:project_path]  = "."
  configs[:http_path]     = "/"
  configs[:sass_path]     = $css_source
  configs[:css_path]      = $css_dest
  configs[:cache_path]    = $cache_dir
  configs[:images_path]   = "static/images/"
  configs[:output_style]  = debug ? :nested : :compressed
  configs[:line_comments] = debug ? true    : false
  Compass.add_configuration(configs, "basho_docs_configs")

  # This will grab every .scss file in the $css_source directory, and run them
  # through Compass, generating equivalently-named .css files in the static/css
  # directory. We should try to keep the number of compiled sources to a minimum
  # though, and simply add new/parsed scss/css files to `all.scss`.
  compiler = Compass.sass_compiler({
    :only_sass_files => Dir.glob("#{$css_source}/*.scss")
  })
  compiler.compile!
end
