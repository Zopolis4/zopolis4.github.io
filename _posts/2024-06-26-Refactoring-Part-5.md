# crew remove

## Know Thy Enemy

Here's the starting point, with some function calls inlined:
```ruby
def remove(pkg_name)
  # make sure the package is actually installed
  unless @device[:installed_packages].any? { |pkg| pkg[:name] == pkg_name } || File.file?(File.join(CREW_META_PATH, "#{pkg_name}.filelist"))
    puts "Package #{pkg_name} isn't installed.".lightred
    return
  end

  # Perform any operations required prior to package removal.
  search pkg_name, true
  @pkg.preremove unless @in_fixup

  # Preserve CREW_ESSENTIAL_FILES and make sure they are real files
  # and not symlinks, because preserving symlinked libraries does not
  # prevent breakage.
  CREW_ESSENTIAL_FILES = `LD_TRACE_LOADED_OBJECTS=1 #{CREW_PREFIX}/bin/ruby`.scan(/\t([^ ]+)/).flatten +
                        %w[libzstd.so.1 libstdc++.so.6]
  CREW_ESSENTIAL_FILES.uniq!
  CREW_ESSENTIAL_FILES.each do |file|
    next unless File.symlink?("#{CREW_LIB_PREFIX}/#{file}")

    canonicalized_file = `readlink -m #{CREW_LIB_PREFIX}/#{file}`.chomp
    if File.file?(canonicalized_file) && canonicalized_file.include?(CREW_PREFIX)
      puts "Replacing symlinked essential #{file} with hard link to #{canonicalized_file} to avoid breakage.".lightblue if @opt_verbose
      FileUtils.ln(canonicalized_file, "#{CREW_LIB_PREFIX}/#{file}", force: true)
    end
  end

  conflicts = []
  if File.file?("#{Dir.pwd}/filelist")
    if File.file?(File.join(CREW_META_PATH, "#{pkg_name}.filelist"))
      puts 'Checking for conflicts with files from installed packages...'.orange
      conflictscmd = `grep --exclude=#{File.join(CREW_META_PATH, "#{pkg_name}.filelist")} --exclude=#{CREW_META_PATH}/\\\*_build.filelist -Fxf #{Dir.pwd}/filelist #{CREW_META_PATH}/*.filelist`
      conflicts = conflictscmd.gsub(/(\.filelist|#{CREW_META_PATH})/, '').split("\n")
      conflicts.reject!(&:empty?)
    end
  elsif File.file?(File.join(CREW_META_PATH, "#{pkg_name}.filelist"))
    puts "Checking for conflicts of #{pkg_name} with files from installed packages...".orange
    conflictscmd = `grep --exclude=#{File.join(CREW_META_PATH, "#{pkg_name}.filelist")} --exclude=#{CREW_META_PATH}/\\\*_build.filelist -Fxf #{File.join(CREW_META_PATH, "#{pkg_name}.filelist")} #{CREW_META_PATH}/*.filelist`
    conflicts = conflictscmd.gsub(/(\.filelist|#{CREW_META_PATH})/, '').split("\n")
    conflicts.reject!(&:empty?)
  end
  if conflicts.any?
    puts 'There is a conflict with the same file in another package:'.orange
    puts conflicts.to_s.orange
  end
  conflicts.map! { |x| x.to_s.partition(':').last}

  # if the filelist exists, remove the files and directories installed by the package
  if File.file?(File.join(CREW_META_PATH, "#{pkg_name}.filelist"))
    Dir.chdir CREW_CONFIG_PATH do
      # remove all files installed by the package
      File.foreach("meta/#{pkg_name}.filelist", chomp: true) do |line|
        # Do not remove essential files which crew (and dependencies)
        # rely on, especially during package upgrades or reinstalls.
        # These essential files are enumerated in const.rb as
        # CREW_ESSENTIAL_FILES.
        if CREW_ESSENTIAL_FILES.include?(File.basename(line))
          puts "Removing #{line} will break crew. It was #{'NOT'.lightred} deleted." if @opt_verbose
        else
          puts "Removing file #{line}".lightred if @opt_verbose
          puts "filelist contains #{line}".lightred if @opt_verbose && !line.include?(CREW_PREFIX)
          if line.start_with?(CREW_PREFIX)
            if conflicts.include?(line)
              puts "#{line} is in another package. It will not be removed during the removal of #{pkg_name}".orange
            else
              FileUtils.rm_rf line
            end
          end
        end
      end

      # remove all directories installed by the package
      File.foreach("meta/#{pkg_name}.directorylist", chomp: true) do |line|
        puts "directorylist contains #{line}".lightred if @opt_verbose && !line.include?(CREW_PREFIX)
        next unless Dir.exist?(line) && Dir.empty?(line) && line.include?(CREW_PREFIX)

        puts "Removing directory #{line}".lightred if @opt_verbose
        FileUtils.rmdir(line)
      end

      # remove the file and directory list
      FileUtils.rm_f Dir["meta/#{pkg_name}.{file,directory}list"]
    end
  end

  # remove from installed packages
  puts "Removing package #{pkg_name}".lightred if @opt_verbose
  @device[:installed_packages].delete_if { |elem| elem[:name] == pkg_name }

  # update the device manifest
  File.write "#{CREW_CONFIG_PATH}/device.json", JSON.pretty_generate(JSON.parse(@device.to_json))

  search pkg_name, true
  @pkg.remove unless @in_fixup

  puts "#{pkg_name.capitalize} removed!".lightgreen
end
```

## Divide && Conquer

With the install check, there's no need to also check for the filelist, since we do it later. Also, let's swap to checking via `.none?`. It's not slower because we only expect one match, and its a little clearer. Oh, and we need to load `device.json` instead of relying on the `@device` instance variable.

```ruby
device_json = JSON.load_file(File.join(CREW_CONFIG_PATH, 'device.json'))

# Make sure the package is actually installed before we attempt to remove it.
if device_json['installed_packages'].none? { |entry| entry['name'] == pkg.name }
  puts "Package #{pkg.name} isn't installed.".lightred
  return
end
```

`@in_fixup` is there to handle the case of removing a package which doesn't have a package file anymore, so we can't call its `preremove` or `remove` methods. I don't like it, though, and now that we're passing the loaded package object directly to remove, we can just create a minimal object with stub methods and pass that, which looks something like this:
```ruby
# Create a minimal Package object and pass it to Command.remove
pkg_object = Package
pkg_object.instance_eval do
  self.name = pkg[:pkg_name]
  def self.preremove; end
  def self.remove; end
end
Command.remove(pkg_object, @opt_verbose)
```

`CREW_ESSENTIAL_FILES`, while perfectly functional, is just not a very nice way of checking that we aren't removing any critical libraries.
Instead of finding the currently loaded libraries at runtime and ensuring they aren't deleted, why don't we just find the packages providing those libraries and refuse to remove them?

```ruby
# Don't remove any of the packages ruby (and thus crew) needs to run.
if %w[gcc_lib glibc gmp ruby zlibpkg zstd].include?(pkg.name)
  puts "Refusing to remove essential package #{pkg.name}.".lightred
  return
end
```

As a bonus, we don't have to mess around with linking files anymore, and we avoid the problems that could come with partially removing libraries.

The conflict detection code only looks terrible because I've inlined the entire function, which has a lot more responsiblities in its original context. All we need it to do here is tell us if a file we're about to remove is provided by any other package. We'll get to the exact implementation of this later.

## Gidorah

There are three main components to the actual removal code:
- The removal of files (Ichi)
- The removal of directories (Ni)
- The removal of the filelist and directorylist (San)

Before we tackle that, though, let's clean up that check for the filelist-- the last thing this block needs is more indentation.

```ruby
# We can't remove a package if we don't have the filelist.
unless File.file?(File.join(CREW_META_PATH, "#{pkg.name}.filelist"))
  puts "Unable to remove package #{pkg.name} as it does not have a filelist.".lightred
  return
end
```

Ooh, an error message! Fancy.

Ichi's certainly a beast to be reckoned with, but the desired behavior isn't too bad.

We've already got the `CREW_ESSENTIAL_FILES` check handled elsewhere, so we can drop that `if/else` and save some indentation while we're at it. Dropping some of the verbosity leaves us with some much cleaner behavior. All that we're doing at this point is removing files as long as they start with `CREW_PREFIX` and aren't found in another package.

```ruby
# Remove all files installed by the package.
File.foreach(File.join(CREW_META_PATH, "#{pkg.name}.filelist"), chomp: true) do |line|
  next unless line.start_with?(CREW_PREFIX)
  if system("grep --exclude #{pkg.name}.filelist -Fxq #{line} ./meta/*.filelist")
    puts "#{line} is in another package. It will not be removed during the removal of #{pkg.name}.".orange
  else
    puts "Removing file #{line}".yellow if verbose
    FileUtils.remove_file line
  end
end
```

Honestly, Ni is already pretty well done-- dropping some extra verbosity is really the only notable thing here.

```ruby
# Remove all directories installed by the package.
File.foreach(File.join(CREW_META_PATH, "#{pkg.name}.directorylist"), chomp: true) do |line|
  next unless Dir.exist?(line) && Dir.empty?(line) && line.include?(CREW_PREFIX)
  puts "Removing directory #{line}".yellow if verbose
  Dir.rmdir line
end
```

Kevin is, well, Kevin.

```ruby
# Remove the file and directory lists.
FileUtils.remove_file File.join(CREW_META_PATH, "#{pkg.name}.filelist")
FileUtils.remove_file File.join(CREW_META_PATH, "#{pkg.name}.directorylist")
```

## Apex

Putting it all together, here's what we get.

```ruby
require 'fileutils'
require 'json'
require_relative '../lib/const'

class Command
  def self.remove(pkg, verbose)
    device_json = JSON.load_file(File.join(CREW_CONFIG_PATH, 'device.json'))

    # Make sure the package is actually installed before we attempt to remove it.
    if device_json['installed_packages'].none? { |entry| entry['name'] == pkg.name }
      puts "Package #{pkg.name} isn't installed.".lightred
      return
    end

    # Don't remove any of the packages ruby (and thus crew) needs to run.
    if %w[gcc_lib glibc gmp ruby zlibpkg zstd].include?(pkg.name)
      puts "Refusing to remove essential package #{pkg.name}.".lightred
      return
    end

    # We can't remove a package if we don't have the filelist.
    unless File.file?(File.join(CREW_META_PATH, "#{pkg.name}.filelist"))
      puts "Unable to remove package #{pkg.name} as it does not have a filelist.".lightred
      return
    end

    # Perform any operations required prior to package removal.
    pkg.preremove

    # Remove the files and directories installed by the package.
    Dir.chdir CREW_CONFIG_PATH do
      # Remove all files installed by the package.
      File.foreach(File.join(CREW_META_PATH, "#{pkg.name}.filelist"), chomp: true) do |line|
        next unless line.start_with?(CREW_PREFIX)
        if system("grep --exclude #{pkg.name}.filelist -Fxq #{line} ./meta/*.filelist")
          puts "#{line} is in another package. It will not be removed during the removal of #{pkg.name}.".orange
        else
          puts "Removing file #{line}".yellow if verbose
          FileUtils.remove_file line
        end
      end

      # Remove all directories installed by the package.
      File.foreach(File.join(CREW_META_PATH, "#{pkg.name}.directorylist"), chomp: true) do |line|
        next unless Dir.exist?(line) && Dir.empty?(line) && line.include?(CREW_PREFIX)
        puts "Removing directory #{line}".yellow if verbose
        Dir.rmdir line
      end

      # Remove the file and directory lists.
      FileUtils.remove_file File.join(CREW_META_PATH, "#{pkg.name}.filelist")
      FileUtils.remove_file File.join(CREW_META_PATH, "#{pkg.name}.directorylist")
    end

    # Remove the package from the list of installed packages in device.json.
    puts "Removing package #{pkg_name} from device.json".yellow if verbose
    device_json['installed_packages'].delete_if { |entry| entry['name'] == pkg.name }

    # Update device.json with our changes.
    File.write File.join(CREW_CONFIG_PATH, 'device.json'), JSON.pretty_generate(JSON.parse(device_json.to_json))

    # Perform any operations required after package removal.
    pkg.remove

    puts "#{pkg.name} removed!".lightgreen
  end
end
```

PR [here](https://github.com/chromebrew/chromebrew/pull/9529).
