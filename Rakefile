CORE_GEMSPEC = Gem::Specification.load('concurrent-ruby.gemspec')
EXT_GEMSPEC = Gem::Specification.load('concurrent-ruby-ext.gemspec')
EXTENSION_NAME = 'concurrent_ruby_ext'

$:.push File.join(File.dirname(__FILE__), 'lib')
require 'extension_helper'

def safe_load(file)
  begin
    load file
  rescue LoadError => ex
    puts 'Error loading rake tasks, but will continue...'
    puts ex.message
  end
end

Dir.glob('tasks/**/*.rake').each do |rakefile|
  safe_load rakefile
end

#desc 'Run benchmarks'
task :bench do
  exec 'ruby -Ilib -Iext examples/bench_atomic.rb'
end

if defined?(JRUBY_VERSION)
  require 'rake/javaextensiontask'

  Rake::JavaExtensionTask.new(EXTENSION_NAME, CORE_GEMSPEC) do |ext|
    ext.ext_dir = 'ext'
  end

elsif Concurrent.allow_c_extensions?
  require 'rake/extensiontask'

  Rake::ExtensionTask.new(EXTENSION_NAME, EXT_GEMSPEC) do |ext|
    ext.ext_dir = "ext/#{EXTENSION_NAME}"
    ext.cross_compile = true
    ext.cross_platform = ['x86-mingw32', 'x64-mingw32']
  end

  ENV['RUBY_CC_VERSION'].to_s.split(':').each do |ruby_version|
    platforms = {
      'x86-mingw32' => 'i686-w64-mingw32',
      'x64-mingw32' => 'x86_64-w64-mingw32'
    }
    platforms.each do |platform, prefix|
      task "copy:#{EXTENSION_NAME}:#{platform}:#{ruby_version}" do |t|
        %w[lib tmp/#{platform}/stage/lib].each do |dir|
          so_file = "#{dir}/#{ruby_version[/^\d+\.\d+/]}/#{EXTENSION_NAME}.so"
          if File.exists?(so_file)
            sh "#{prefix}-strip -S #{so_file}"
          end
        end
      end
    end
  end
else
  task :clean
  task :compile
end

Rake::Task[:clean].enhance do
  rm_rf 'pkg/classes'
  rm_rf 'tmp'
  rm_rf 'lib/1.9'
  rm_rf 'lib/2.0'
  rm_f Dir.glob('./lib/*.jar')
  rm_f Dir.glob('./**/*.bundle')
  mkdir_p 'pkg'
end

begin
  require 'rspec'
  require 'rspec/core/rake_task'

  RSpec::Core::RakeTask.new(:spec) do |t|
    t.rspec_opts = '--color --backtrace --format documentation'
  end

  task :default => [:clean, :compile, :spec]
rescue LoadError
  puts 'Error loading Rspec rake tasks, probably building the gem...'
end

namespace :build do
  
  build_deps = [:clean]
  build_deps << :compile if defined?(JRUBY_VERSION)

  desc 'Build the concurrent-ruby gem'
  task :core => build_deps do
    sh "gem build #{CORE_GEMSPEC.name}.gemspec"
    sh 'mv *.gem pkg/'
    Rake::Task[:clean].execute
  end

  if Concurrent.allow_c_extensions?
    desc 'Build the concurrent-ruby-ext gem'
    task :ext => [:clean, :compile] do
      sh "gem build #{EXT_GEMSPEC.name}.gemspec"
      sh 'mv *.gem pkg/'
      Rake::Task[:clean].execute
    end
  else
    task :ext
  end
end

desc 'Build all gems for this platform'
task :build => ['build:core', 'build:ext']
