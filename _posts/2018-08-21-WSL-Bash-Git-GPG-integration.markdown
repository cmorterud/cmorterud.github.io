---
layout: post
title:  "Integrating GPG with Git and Bash on Windows Subsystem for Linux"
date:   2018-08-21 13:00:00 -0400
categories: development
---
<!-- You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets: -->
<!-- 
I have been a longtime macOS user since I started college, staying
as far away from Windows as I could. That was mainly due to my college's use
of Red Hat Enterprise Linux and macOS's command line utilities and Homebrew
out of the box being
a near approximation for RHEL's utilities.  -->

Over the course of 3 years studying Computer Science, I have made
heavy use of Bash and Git. Just last semester, I took a course called
Introduction to Cryptography where I learned about secure systems,
cryptographic schemes, authentication, abstract algebra, and more.
I was introduced to GnuPG as a program that abstracts away all
but the most essential details of encryption, and I was very impressed
with the capabilities and power that GnuPG provides to the user. 

Let's say you are interested in signing your Git commits. There is much
more that can be signed, but signing Git commits can be automated once
set up, and offers some additional assurance to others that you, yourself
are the author of some commits.

I will start off by making sure GPG and Git
are installed on Bash for Ubuntu WSL.

{% highlight bash %}
$ sudo apt-get install gnupg --yes
$ sudo apt-get install git --yes
# --yes is to automatically insert yes at prompts
{% endhighlight %}

To generate a key, type

{% highlight bash %}
$ gpg --gen-key

Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
Your selection?
{% endhighlight %}

I select 1 for RSA, because it's my favorite and I have implemented
it before.

{% highlight bash %}
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048)
{% endhighlight %}

I select 4096 bits for the maximum amount possible of security, at the
cost of added time to decrpyt and encrypt, although 2048 bits is
still perfectly valid on 8/21/2018.

{% highlight bash %}
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0)
{% endhighlight %}

I hit enter to select "key does not expire," although, one could
choose to have the key become invalid (untrusted) in a certain amount of time.
If you choose to have the key expire in a certain amount of time,
then you will need to tell git about the new key when used to  automatically 
sign 
commits and/or tags, which we will talk about later in this post.

{% highlight bash %}
You need a user ID to identify your key; the software constructs the user ID
from the Real Name, Comment and Email Address in this form:
    "Heinrich Heine (Der Dichter) <heinrichh@duesseldorf.de>"

Real name: Jane Smith
Email address: JaneSmith@example.com
Comment: Example key
{% endhighlight %}

You will next fill in your real name, email address, and an optional comment.

The next prompt will be for a passphrase AKA a password to access the key.
Keep in mind that if your key falls into the wrong hands, the passphrase
is the only defense against impersonation, but if you have a passphrase,
you will need to type it in at each commit if you automatically sign commits.
I didn't make a passphrase because I trust that my computer is secure,
and that no one else is able to access it.

{% highlight bash %}
You need a Passphrase to protect your secret key.

You don't want a passphrase - this is probably a *bad* idea!
I will do it anyway.  You can change your passphrase at any time,
using this program with the option "--edit-key".
{% endhighlight %}

Next, GPG will generate pseudo-random bytes to generate a
pair of keys.

{% highlight bash %}
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
......+++++
.....+++++
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
........+++++
...........f+++++
gpg: key 1234ABCD marked as ultimately trusted
public and secret key created and signed.
{% endhighlight %}

Boom! You now have generated a pair of keys that can authenticate
yourself to others, as well as encrypt communications that are sent to you.

{% highlight bash %}
# show your public keyring (contains public keys)
$ gpg --list-keys
pub   4096R/1234ABCD 2018-08-21
uid                  Jane Smith (Example key) <JaneSmith@example.com>
sub   4096R/QWERTYUI 2018-08-21
{% endhighlight %}

Now that you have a public and private key, you can sign commits. To sign
commits, you will need to specify the key you are using, as well
as to automatically sign commits as they are made.

{% highlight bash %}
# set this key as your global signing key (use for all repositories)
$ git config --global user.signingkey 1234ABCD
# tell Git to attempt to sign your commits automatically (for all repositories)
$ git config --global commit.gpgsign true
{% endhighlight %}

Next time you make a commit, Git will try to sign the commit, and will
tell if you if there was an issue with signing the commit.

You might be interested in hosting your projects on Github, so there
is a way to get that niced "Verified" checkmark next to your commits,
and that is by telling Github what your public key is, so Github
can verify the signature on your commits.

You can do this by heading over to https://github.com/settings/profile and
clicking on SSH and GPG keys. There, you can click on "add GPG key,"
and there will be a box to paste the ASCII representation of your public key.

{% highlight bash %}
$ gpg --export --armor 1234ABCD
-----BEGIN PGP PUBLIC KEY BLOCK-----

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
A++0YGCjliqd39fOixrB
=Cczl
-----END PGP PUBLIC KEY BLOCK-----
{% endhighlight %}

You need to paste that huge chunk of text into the textbox given on 
Github, and then if everything works out, you can start
signing commits and getting the nice looking "Verified" stamps
in your commit history.

Thanks for reading! Please let me know if there are any questions
at {{ site.email }}

