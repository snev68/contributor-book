# Nu Contributor Book

You can view it live at [www.nushell.sh/contributor-book/](https://www.nushell.sh/contributor-book/)

This is a draft of the Nushell Contributor Book. It attempts to cover the basics of how Nu works internally, to better enable contributors to have a solid understanding of Nu and how best to contribute.

Please see the directory for each language for a full list of available chapters.

## Contributing

New contributors are welcomed! We are thankful for your help.

The best way to get started is looking around Nu's source code for the things not covered here. If you have questions or would like to pair with another nuño, head over to [Nu Community Discord](https://discord.gg/NtAbbGn). Contributors are encouraged to reach out and open issues to propose any changes before investing time.

## Running locally with Ruby

To get started installing dependencies.

``` sh
git clone https://github.com/nushell/contributor-book.git
cd contributor-book
bundle
bundle exec jekyll serve
```

Then visit [http://localhost:4000/](http://localhost:4000/)

### Requirements

If the `bundle` command is not found, you can install it with `gem install bundler`.

If the `bundle` command fails to run, you may need to upgrade your Ruby version. You can use [RVM](https://rvm.io/) to install it:

``` sh
curl -L https://get.rvm.io | bash -s stable
rvm install $(cat .ruby-version)
rvm use $(cat .ruby-version)
```

### Troubleshooting tips for Windows

For Windows developers using [RubyInstaller](http://rubyinstaller.org/), you'll need to [download the DevKit](http://rubyinstaller.org/downloads) and install it using instructions:
<https://github.com/oneclick/rubyinstaller/wiki/Development-Kit>

After you have a proper install, you can then run:

``` sh
gem install bundler wdm tzinfo-data
gem update listen jekyll
```

Once Jekyll comes up, you'll be prompted by Windows Firewall. Click "Allow access" and you'll be in business!
