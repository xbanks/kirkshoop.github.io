require 'rubygems'
require 'rake'
require 'rdoc'
require 'date'
require 'yaml'
require 'tmpdir'
require 'jekyll'

desc "Serve blog locally http://localhost:4000"
task :serve do
    system "git checkout source"
    system "bundle exec jekyll serve --watch --drafts"
end

desc "Generate blog files"
task :generate do
  Jekyll::Site.new(Jekyll.configuration({
    "source"      => ".",
    "destination" => "_site"
  })).process
end


desc "Generate and publish blog to master"
task :publish => [:generate] do
  Dir.mktmpdir do |tmp|
    system "git add --all"
    system "git add -u"
    system "git status"
    system "git stash"
    system "mv _site/* #{tmp}"
    system "git checkout -B master"
    system "rm -rf *"
    system "mv #{tmp}/* ."
    message = "Site updated at #{Time.now.utc}"
    system "git add ."
    system "git commit -am #{message.shellescape}"
    system "git push origin master --force"
    system "git checkout source"
    system "git stash pop --index"
    system "echo yolo"
  end
end

task :default => :publish