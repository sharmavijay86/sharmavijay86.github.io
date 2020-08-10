
# How to install Ubuntu Server 18.04 Desktop GUI and XRDP

###Pre-run system upgrade:
```
sudo apt-get update && sudo apt-get -y upgrade
```
 
1. Install minimal Ubuntu Desktop (default Desktop of Ubuntu 18)

```
sudo apt install gnome-session gdm3 gnome-tweak-tool gnome-software synaptic
sudo reboot
```
 
2. Install XRDP server,

```
sudo apt-get -y install xrdp
```
 
3. Enable port 3389 for remote desktop connection if the port 3389 is not enabled.

```
sudo ufw allow 3389/tcp
```
 
4. Create a polkit configuration file, 

```
sudo nano /etc/polkit-1/localauthority.conf.d/02-allow-colord.conf
```
   input the following content,

```
polkit.addRule(function(action, subject) {
if ((action.id == “org.freedesktop.color-manager.create-device” || action.id == “org.freedesktop.color-manager.create-profile” || action.id == “org.freedesktop.color-manager.delete-device” || action.id == “org.freedesktop.color-manager.delete-profile” || action.id == “org.freedesktop.color-manager.modify-device” || action.id == “org.freedesktop.color-manager.modify-profile”) && subject.isInGroup(“{group}”))
{
return polkit.Result.YES;
}
});
```
 
5. Restart XRDP

```
sudo /etc/init.d/xrdp restart
```
 
Now you can connect the Ubunut Desktop through remote desktop connection!!
