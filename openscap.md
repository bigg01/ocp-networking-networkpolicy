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


# running job in OCP

https://github.com/evgenyz/openscap-ocp
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: oscap
spec:
  parallelism: 1
  completions: 1
  activeDeadlineSeconds: 600
  backoffLimit: 2
  template:
    metadata:
      name: oscap
    spec:
      containers:
      - name: oscap
        image: docker.io/ekolesni/openscap-ocp4
        command: ["oscap-chroot", "/host", "--verbose", "DEVEL", "xccdf", "eval", "--fetch-remote-resources", "--profile", "xccdf_org.ssgproject.content_profile_ospp",  "--report", "/tmp/report.html", "/var/lib/content/ssg-fedora-ds-1.3.xml"]
        securityContext:
          privileged: true
          runAsUser: 0
        volumeMounts:
        - mountPath: /host
          name: host
      hostNetwork: true
      hostPID: true
#      nodeName: ip-10-0-129-101.ec2.internal
      restartPolicy: Never
      volumes:
      - hostPath:
        path: /
        type: Directory
        name: host
```

tekton example:

https://github.com/kabanero-io/kabanero-security/blob/399064f16265f9a16960602c70c6366bdd98dd8f/pipelines/samples/scan-pipeline.yaml

https://github.com/kabanero-io/kabanero-security/tree/master/pipelines/samples/images/scanner


## generate guide
oscap xccdf generate guide --profile rht-ccp \
  --cpe /usr/share/xml/scap/ssg/content/ssg-rhel7-cpe-dictionary.xml \
        /usr/share/xml/scap/ssg/content/ssg-rhel7-xccdf.xml > /var/www/html/security_guide.html
        
        
 ## custom stylesheet
oscap xccdf generate custom --stylesheet /vagrant/vendor/govready/prototypes/openscap/xsl/xccdf-report.xsl --output /var/www/govready-html/gr-xccdf.html /var/www/govready-html/usgcb-rhel6-server.xml
