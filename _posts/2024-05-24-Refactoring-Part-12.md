# Refactoring Part 12: crew search

## The Setup

Here's what I was dealing with:
```ruby
def regexp_search(pkg_pat)
  re = Regexp.new(pkg_pat, true)
  results = Dir["#{CREW_PACKAGES_PATH}/*.rb"] \
            .select  { |f| File.basename(f, '.rb') =~ re } \
            .each    { |f| print_package(f, CREW_VERBOSE) }
  if results.empty?
    Dir["#{CREW_PACKAGES_PATH}/*.rb"].each do |package_path|
      package_name = File.basename package_path, '.rb'
      begin
        set_package package_path
      rescue StandardError => e
        puts "Error with #{pkg_name}.rb: #{e}".red unless e.to_s.include?('uninitialized constant')
      end
      if @pkg.description =~ /#{pkg_pat}/i
        print_current_package CREW_VERBOSE
        results.push(package_name)
      end
    end
  end
  abort "Package #{pkg_pat} not found. :(".lightred if results.empty?
end

def search_command(args)
  args['<name>'].each do |name|
    regexp_search name
  end.empty? && list_packages
end
```

Oh, and here are the functions it calls.
```ruby
def print_package(pkg_path, extra = false)
  pkg_name = File.basename pkg_path, '.rb'
  begin
    set_package pkg_path
  rescue StandardError => e
    warn "Error with #{pkg_name}.rb: #{e}".red unless e.to_s.include?('uninitialized constant')
  end
  print_current_package extra
end

def print_current_package(extra = false)
  status = if PackageUtils.installed?(@pkg.name)
             :installed
           elsif !PackageUtils.compatible?(@pkg)
             :incompatible
           else
             :available
           end
  case status
  when :installed
    print @pkg.name.lightgreen
  when :incompatible
    print @pkg.name.lightred
  when :available
    print @pkg.name.lightblue
  end
  print ": #{@pkg.description}".lightblue if @pkg.description
  if extra
    puts ''
    puts @pkg.homepage if @pkg.homepage
    puts "Version: #{@pkg.version}"
    print "License: #{@pkg.license}" if @pkg.license
  end
  puts ''
end

def set_package(pkg_path)
  begin
    @pkg = Package.load_package(pkg_path)
  rescue SyntaxError => e
    warn "#{e.class}: #{e.message}".red
  end
  @pkg.build_from_source = true if @opt_recursive
end

def list_packages
  Dir["#{CREW_PACKAGES_PATH}/*.rb"].each do |filename|
    print_package filename
  end
end
```
## Before We Start

Let's break down what this code actually does.

1. If the input string is empty, print the name of every package.
2. Otherwise, convert the input string into a case-insensitive regular expression.
3. Take the regular expression and checks it against the filename of every package.
   1. If it matches, load the package, rescuing any errors, then print the name and description.
      - If the package is installed, print the name in green.
      - If it is incompatible with the current architecture, print the name in red.
      - If it is compatible but not installed, print the name in blue.
   2. If the `--verbose` argument was passed, also print the homepage, version and license of the package.
      - Check that we have a value for the homepage and the license before we print them.
4. If there weren't any matches, load each package, rescuing any errors, and check the regex against the description.
   1. If it matches, load the package, rescuing any errors, then print the name and description.
      - If the package is installed, print the name in green.
      - If it is incompatible with the current architecture, print the name in red.
      - If it is compatible but not installed, print the name in blue.
   2. If the `--verbose` argument was passed, also print the homepage, version and license of the package.
      - Check that we have a value for the homepage and the license before we print them.
5. If we still don't have any matches, abort and tell the user we couldn't find anything.


I find this rubber duck refactoring helpful because it allows you to consider the behaviour of the code, not just the implementation.

If we were just considering the implementation, there's still plenty of opportunities to clean things up-- no need to load packages twice, or to convert the input string into a regex twice, for example.

## Thinking Hard

But, in this case, its clear that only searching the description if the name search turns up nothing is significantly increasing the complexity.

And, for that matter, its not particularly intuitive behaviour either. APT doesn't do it, and taking a look at the help for `crew search` makes the shortcomings of this approach obvious.
```
crew search audio".lightblue + " will display all packages with `audio` in the name.
crew search | grep -i audio".lightblue + " will display all packages with `audio` in the name or description.
```

Abandoning this approach immediately opens the door to a much more reasonable program structure.

## Hardly Thinking

```ruby
Dir["#{CREW_PACKAGES_PATH}/*.rb"].each do |package_path|
  pkg = Package.load_package(package_path)
  # Create a case-insensitive regex from the passed string.
  regex = Regexp.new(regex_string, true)
  next unless regex.match?(File.basename(package_path, '.rb')) || regex.match?(pkg.description)
  print_package package_path, CREW_VERBOSE
end
```

We've also dropped the error message if no results were found, eliminating the need for any array shenanigans.

But we are splitting this out into a file after all, so we can't rely on any functions from `bin/crew`, so `print_package` needs to go.

As a reminder:
```ruby
def print_package(pkg_path, extra = false)
  pkg_name = File.basename pkg_path, '.rb'
  begin
    set_package pkg_path
  rescue StandardError => e
    warn "Error with #{pkg_name}.rb: #{e}".red unless e.to_s.include?('uninitialized constant')
  end
  print_current_package extra
end

def print_current_package(extra = false)
  status = if PackageUtils.installed?(@pkg.name)
             :installed
           elsif !PackageUtils.compatible?(@pkg)
             :incompatible
           else
             :available
           end
  case status
  when :installed
    print @pkg.name.lightgreen
  when :incompatible
    print @pkg.name.lightred
  when :available
    print @pkg.name.lightblue
  end
  print ": #{@pkg.description}".lightblue if @pkg.description
  if extra
    puts ''
    puts @pkg.homepage if @pkg.homepage
    puts "Version: #{@pkg.version}"
    print "License: #{@pkg.license}" if @pkg.license
  end
  puts ''
end
```

We've already loaded the package, so we can skip that and merge the two functions:
```ruby
def print_current_package(extra = false)
  status = if PackageUtils.installed?(pkg.name)
             :installed
           elsif !PackageUtils.compatible?(pkg)
             :incompatible
           else
             :available
           end
  case status
  when :installed
    print pkg.name.lightgreen
  when :incompatible
    print pkg.name.lightred
  when :available
    print pkg.name.lightblue
  end
  print ": #{pkg.description}".lightblue if pkg.description
  if extra
    puts ''
    puts pkg.homepage if pkg.homepage
    puts "Version: #{pkg.version}"
    print "License: #{pkg.license}" if pkg.license
  end
  puts ''
end
```

That `status` variable is just begging to go:
```ruby
def print_current_package(extra = false)
  if PackageUtils.installed?(pkg.name)
    print pkg.name.lightgreen
  elsif !PackageUtils.compatible?(pkg)
    print pkg.name.lightred
  else
    print pkg.name.lightblue
  end

  print ": #{pkg.description}".lightblue if pkg.description
  if extra
    puts ''
    puts pkg.homepage if pkg.homepage
    puts "Version: #{pkg.version}"
    print "License: #{pkg.license}" if pkg.license
  end
  puts ''
end
```

I don't know about you, but I'm pretty sure we can do better than `puts ''`:
```ruby
def print_current_package(extra = false)
  if PackageUtils.installed?(pkg.name)
    print pkg.name.lightgreen
  elsif !PackageUtils.compatible?(pkg)
    print pkg.name.lightred
  else
    print pkg.name.lightblue
  end

  puts ": #{pkg.description}".lightblue if pkg.description
  if extra
    puts pkg.homepage if pkg.homepage
    puts "Version: #{pkg.version}"
    puts "License: #{pkg.license}" if pkg.license
  end
end
```

Oh, and we know the package will have a description (and a homepage, version and so on) thanks to [the test I implemented for that a while back:](https://github.com/chromebrew/chromebrew/pull/8987)

```ruby
def print_current_package(extra = false)
  if PackageUtils.installed?(pkg.name)
    print pkg.name.lightgreen
  elsif !PackageUtils.compatible?(pkg)
    print pkg.name.lightred
  else
    print pkg.name.lightblue
  end
  puts ": #{pkg.description}".lightblue

  if extra
    puts pkg.homepage
    puts "Version: #{pkg.version}"
    puts "License: #{pkg.license}"
  end
end
```

Now, lets put this back into the function:
```ruby
Dir["#{CREW_PACKAGES_PATH}/*.rb"].each do |package_path|
  pkg = Package.load_package(package_path)
  # Create a case-insensitive regex from the passed string.
  regex = Regexp.new(regex_string, true)
  next unless regex.match?(File.basename(package_path, '.rb')) || regex.match?(pkg.description)
  if PackageUtils.installed?(pkg.name)
    print pkg.name.lightgreen
  elsif !PackageUtils.compatible?(pkg)
    print pkg.name.lightred
  else
    print pkg.name.lightblue
  end
  puts ": #{pkg.description}".lightblue

  if CREW_VERBOSE
    puts pkg.homepage
    puts "Version: #{pkg.version}"
    puts "License: #{pkg.license}"
  end
end
```

Oh right, this is a loop:
```ruby
Dir["#{CREW_PACKAGES_PATH}/*.rb"].each do |package_path|
  pkg = Package.load_package(package_path)
  # Create a case-insensitive regex from the passed string.
  regex = Regexp.new(regex_string, true)
  next unless regex.match?(File.basename(package_path, '.rb')) || regex.match?(pkg.description)
  if PackageUtils.installed?(pkg.name)
    print pkg.name.lightgreen
  elsif !PackageUtils.compatible?(pkg)
    print pkg.name.lightred
  else
    print pkg.name.lightblue
  end
  puts ": #{pkg.description}".lightblue

  next unless CREW_VERBOSE
  puts pkg.homepage
  puts "Version: #{pkg.version}"
  puts "License: #{pkg.license}"
end
```

## Wrapping Up

And here's the finished product in `commands/search.rb`:
```ruby
require_relative '../lib/const'
require_relative '../lib/package'
require_relative '../lib/package_utils'

class Command
  def self.search(regex_string, verbose)
    Dir["#{CREW_PACKAGES_PATH}/*.rb"].each do |package_path|
      pkg = Package.load_package(package_path)
      # Create a case-insensitive regex from the passed string.
      regex = Regexp.new(regex_string, true)
      next unless regex.match?(File.basename(package_path, '.rb')) || regex.match?(pkg.description)
      # Installed packages have green names, incompatible packages have red, and compatible but not installed have blue.
      if PackageUtils.installed?(pkg.name)
        print pkg.name.lightgreen
      elsif !PackageUtils.compatible?(pkg)
        print pkg.name.lightred
      else
        print pkg.name.lightblue
      end
      puts ": #{pkg.description}".lightblue

      next unless verbose
      puts pkg.homepage
      puts "Version: #{pkg.version}"
      puts "License: #{pkg.license}"
    end
  end
end
```

And calling it from `bin/crew`:
```ruby
def search_command(args)
  args['<name>'].each do |name|
    Command.search(name, CREW_VERBOSE)
  end
end
```

PR [here](https://github.com/chromebrew/chromebrew/pull/9833).
