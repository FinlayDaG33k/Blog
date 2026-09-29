# MikroTik: User Manager, WiFi and VLANs

Sometimes, you want to have more control over the users that you have on your WiFi.  
This could be due to running a business with multiple distinct roles or just because you're pedantic about your parents' unknown wifi devices.  
While it can be done using something like FreeRADIUS, doing so can be a lot more complicated, difficult to maintain and just be outright overkill for WiFi.
Luckily for us, MikroTik has released a very nifty tool called "UserManager" that helps us tackle a lot of the complexity.  
Additionally, when using MikroTik's APs, you can even setup per user (or per-group) VLAN assignments.  
This way, you do not need to setup 1337666 different SSIDs, which clutter the airspace and make it harder for users to use.
And finally, each user has their own login credentials, which means you can deny access more easily (by just disabling a user) and it also makes it harder for people to just sniff packets over the air, as they will all have their own encryption keys!

Reasons to use this setup:

- You want to isolate certain users/groups without making 1337666 separate SSIDs.
- You want to easily see which devices belongs to which person.
- You want to quickly be able to revoke access for certain people.

Reasons to not use this setup:

- You only want to isolate IoT or guest devices.
- You run silly consumer electronics that do not support WPA(2)-EAP (eg. a Nintendo Switch or a printer).

**NOTE**: This setup only handles setting up User Manager, your WiFi and assigning VLANs to users.  
It does not handle setting up WiFi, CAPsMAN or VLANs on routers, switches and APs themselves.

## Installing User Manager

First, we'll need to install the required package for User Manager.  
At the time of writing, this package is sub-500KiB, so it should fit on nearly all devices.  

Let's update the package cache first:
```
/system package update check-for-updates without-paging
```

It may show the following output:
``` 
channel: stable                       
installed-version: 7.19                         
status: finding out latest version...

channel: stable                  
installed-version: 7.19                    
latest-version: 7.20.2                  
status: New version is available
```

I'll be skipping updating the RouterOS version for now, but you can do so yourself if you want to.  

Next, check that the User Manager package gets listed:

``` 
/system package print
```

Which should show something like the following output:

``` 
Flags: X - DISABLED; A - AVAILABLE
Columns: NAME, VERSION, BUILD-TIME, SIZE
#    NAME            VERSION  BUILD-TIME           SIZE     
0    routeros        7.19     2025-05-22 07:53:44  12.4MiB
1 XA user-manager                                  336.1KiB
```

If so, we can enable it:

```
/system package enable user-manager
```

Then reboot the device and make sure User Manager is installed after it has come back online.  

``` 
/system reboot
/system package print
```

It should now show the following:

``` 
Flags: X - DISABLED; A - AVAILABLE
Columns: NAME, VERSION, BUILD-TIME, SIZE
 #    NAME            VERSION  BUILD-TIME           SIZE     
 0    routeros        7.19     2025-05-22 07:53:44  12.4MiB
 1    user-manager    7.19     2025-05-22 07:53:44  336.1KiB
```

If this is the case, then User Manager is installed!

## (Optionally) Move the database to a USB disk

If your RouterOS device has a USB port (or M.2 ports or U.2 ports etc. etc.), you can opt to move the User Manager database there.  
Doing so saves NAND cycles and while some will argue that it isn't that bad or that they will still last for ages, I personally prefer just using a USB drive (which is why I run my User Manager on an old `hAP AC Lite` instead of my `CRS317`).  
This of course also helps if you either re-purpose an older device with limited storage (like the aforementioned `hAP AC Lite`) as the database can get to 4MB reasonably fast, which on 16MB of NAND, is a lot!  
I assume you have already mounted your storage, if not, you'll need to figure that out first.

After that, you can tell User Manager to use a different path for its database.  
I'll be putting it on `usb1` in a directory `user-manager`.  
You an put it in the root if you want, I prefer this style of organization as it makes migrations and backups a lot easier.

```
/user-manager database
  db-path=usb1/user-manager5
```

## Adding a User

For this user, we'll assume you're gonna use `VLAN 1000`.  
If you want to use a different VLAN, change the `Mikrotik-Wireless-VLANID` attribute's value accordingly.  

```
/user-manager user
  add attributes=Mikrotik-Wireless-VLANID:1000 name=someuser shared-users=unlimited
```

## Updating your WiFi's security settings

First, we need to allow your router to access User Manager.
```
/user-manager router
  add address=127.0.0.1 name="localhost" shared-secret=lamesecret
```

And then we need to tell our device to use the RADIUS server..
``` 
/radius
  add address=127.0.0.1 require-message-auth=no service=wireless secret=lamesecret
```

**NOTE**: If you run User Manager on a different device than what handles WiFi authentication, change the addresses accordingly.  

**NOTE**: The secrets must be the same on the User Manager as well as the RADIUS config.  
You can leave this empty but it's not recommended.

## (Optionally) Remove User Manager Sessions

After a while, old User Manager sessions can start to accumulate.  
As such, you can add a scheduler that will clean out the old sessions every week.  
You can do it more often, or less often if you desire by changing the `interval`.

``` 
/system scheduler
  add interval=1w name="Userman Session Clean" on-event="/user-manager/session remove [find where active=no]" policy=read,write start-date=1970-01-01 start-time=00:00:0
```

## (Optionally) Set Auth methods

To make things easier for myself when setting up a new device, you can setup different auth methods.  
By default, MikroTik has set all `Outer Auths` and all `Inner Auths` to be enabled.  
This means that, when asked for your credentials, you have to manually select the correct ones, which is annoying and increases chance of mistakes.

However, since we do not use certificate-based authentication, we only really need `EAP TTLS` for the `Outer Auth` and `TTLS PAP` for the `Inner Auth` to be enabled.  
What this means is that first, the client will create TLS tunnel (but unlike `EAP TLS`, doesn't require a client-side certificate) to the UserManager and then use `PAP` to actually authenticate with the RADIUS server.  
This _does_ create more steps in authentication but makes it plenty secure for home use.

To do this, we need to update the default profile (you can also create a new one if you prefer that):

```
/user-manager user group
  set [ find default-name=default ] inner-auths=ttls-pap outer-auths=eap-ttls
```

After this, clients will automatically be presented with the right options.

**NOTE**: `EAP TTLS` with `TTLS PAP` should *not* be used in a business or enterprise environment as it sacrifices some security for convenience.
I only use them here because it does not require a PKI (making it easier to deploy on devices I do not control).  
Use `EAP TLS` if you can use certificate-based authentication.

## Final Thoughts

This setup has served me well for years.  
From allowing me to more easily identify a compromised device trying to get into my router, to revoking my ex's wifi access when she left me.  
It has been stable and for the most part, not caused me any issues.

Having used FreeRADIUS in the past for this, I think MikroTik really did well in making UserManager.  
While FreeRADIUS is way more powerful and offers a lot more features, if you want a simple RADIUS server for your home, small business, student dorms etc. etc., then UserManager shouldn't disappoint either!
