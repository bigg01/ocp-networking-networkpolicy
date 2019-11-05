https://github.com/ComplianceAsCode/content/blob/master/docs/manual/user_guide.adoc

# get nightly build
```
wget https://jenkins.complianceascode.io/view/SCAP%20Security%20Guide/job/scap-security-guide-nightly-zip/lastSuccessfulBuild/artifact/scap-security-guide-nightly.zip

unzip bla.zip

ls -lart scap-security-guide-*/*ocp*
-rw-------  1 guo  staff   537146  5 Nov 15:05 scap-security-guide-0.1.47/ssg-ocp4-ds.xml
-rw-------  1 guo  staff  1054417  5 Nov 15:05 scap-security-guide-0.1.47/ssg-ocp3-ds.xml
-rw-------  1 guo  staff   537146  5 Nov 15:05 scap-security-guide-0.1.47/ssg-ocp4-ds-1.2.xml
-rw-------  1 guo  staff  1054417  5 Nov 15:05 scap-security-guide-0.1.47/ssg-ocp3-ds-1.2.xml
```
https://github.com/ComplianceAsCode/content/releases


```sh
 docker run -it openscap/openscap:latest sh
sh-4.4# ls -larht /usr/bin/oscap*
-rwxr-xr-x 1 root root 4.4K Nov 13  2017 /usr/bin/oscap-vm
-rwxr-xr-x 1 root root 9.7K Nov 13  2017 /usr/bin/oscap-ssh
-rwxr-xr-x 1 root root 3.1K Nov 13  2017 /usr/bin/oscap-chroot
-rwxr-xr-x 1 root root 118K Nov 14  2017 /usr/bin/oscap
-rwxr-xr-x 1 root root  23K Jan 16  2018 /usr/bin/oscapd-evaluate
-rwxr-xr-x 1 root root  29K Jan 16  2018 /usr/bin/oscapd-cli
-rwxr-xr-x 1 root root 2.7K Jan 16  2018 /usr/bin/oscapd
```

filesystem scanner - oscap-chroot: Mount a filesystem to the container and run the scan: 
```sh
docker run --rm -v /:/mnt/root -v $(pwd):/mnt/results openscap/openscap:f27-1 oscap-chroot /mnt/root xccdf eval --report /mnt/results/results.html --profile common /usr/share/xml/scap/ssg/content/ssg-fedora-ds.xml
```
