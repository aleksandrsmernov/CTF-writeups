Neighbour

Target IP: 10.114.169.142

The room is supposed to be very easy and can be completed in about 10 minutes.

First, I checked whether the target had a web server running on port 80:

10.114.169.142:80

There was a website with a login page in index.php.

On the page, there was an interesting message:

"Don't have an account? Use the guest account! (Ctrl+U)"

So I tried to log in using the guest account:

guest@guest

The login was successful.

After logging in, I started looking for something interesting. The profile page contained a user parameter in the URL:

http://10.114.169.142/profile.php?user=guest

This looked interesting because the application was directly using the username as an object reference.

Since the room was about IDOR, I tried changing the user parameter from guest to admin:

http://10.114.169.142/profile.php?user=admin

The server did not properly check whether I had permission to access the admin profile.

The page returned:

Hi, admin. Welcome to your site. The flag is: flag{66be95c478473d91a5358f2440c7af1f}

Flag:

flag{66be95c478473d91a5358f2440c7af1f}

The vulnerability was an IDOR (Insecure Direct Object Reference), because I could change the user parameter and access another user's profile without proper authorization.
