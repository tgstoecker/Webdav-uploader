# Python3 WebDAV uploader

Base version copied from here: [https://git.fh-aachen.de/embeddedutils/webdav_uploader/-/tree/main/](https://git.fh-aachen.de/embeddedutils/webdav_uploader/-/tree/main/)

Added minor modifications of my own.

---

Simple uploader (blocking mode) for WebDAV. used to upload files to any WebDAV server (e.g. sciebo) based on the python package [webdavclient3](https://github.com/ezhov-evgeny/webdav-client-python-3). 

## How to use

Example uploading a file 

- Server: https://fh-aachen.sciebo.de/remote.php/webdav
- username: jh874ke
- password: my_password
- remote_path: upload/path/sciebo
- local_path: my_local_file.txt

```bash
python3 webdav_uploader.py -s https://fh-aachen.sciebo.de/remote.php/webdav -u jh874ke -p my_password -r upload/path/sciebo -l my_local_file.txt
```

## Arent there already other WebDAV tools?

As far as i know there are two tools to use WebDAV in linux: [cadaver](http://www.webdav.org/cadaver/) which is a command-line interface (cli) and [davf2s](https://savannah.nongnu.org/projects/davfs2) which allow to mount WebDAV servers as a directory. The difference from these two tools is that this script allows to upload files to a WebDAV server in blocking mode and does not require user interaction as the credentials and folder/files that want to be uploaded are passed as arguments once.

What does this mean? It means that the script will upload the desired directory/file to a WebDAV server without any further user interaction and wait until the upload is finished. As far as I have tried with cadaver,I havent figured how to pass interactive arguments through terminal. I have tried for example:

```bash
echo -e "username\npass\nCommand1\nCommand2\netc"| cadaver https://theserver.com 
```

but after executing it just logins and closes the session without accepting the rest of the interactive commands `"\nCommand1\nCommand2\netc"`.  The other option is [davfs2](https://github.com/volga629/davfs2) but it works [asynchronously](https://serverfault.com/questions/404653/get-webdav-uploading-progress-and-status-linux-davfs2), so when the remote server is mounted, a change or upload in file will not wait until completion. 

Why is it important to upload files in blocking mode? For my objective, I want to upload release files in a Gitlab CI and make sure the job wait until the upload is finished correctly. As mentioned davfs2 does not work in blocking mode and I havent been able to use cadaver in non-interactive mode.

## How to integrate with Gitlab CI?

If you want to integrate this script to your Gitlab CI jobs you require a docker container with:

- git
- python3
- python module [webdavclient3](https://pypi.org/project/webdavclient3/) 

Alternatively you can use [my docker container](https://git.fh-aachen.de/embeddedtools/davfs2/) which already has the necessary requirements.

Example of how to use the python script in a GitlabCI job
```yaml
Create release:
    image: registry.git.fh-aachen.de/embeddedtools/davfs2
    tags: 
    - shared
    stage: release
    script:
    - zip -r my_release.zip /the_release_files -x #Compress release files in a zip
    - git clone https://git.fh-aachen.de/iaam_utils/webdav_uploader #Get the webdav uploader script
    #Make the job upload the zip file and wait until completion
    - python3 webdav_uploader/webdav_uploader.py -s https://fh-aachen.sciebo.de/remote.php/webdav -u $WEBDAV_USER -p $WEBDAV_PASS -r $UPLOAD_PATH -l my_release.zip
    #Note: For security reasones the variables WEBDAV_USER,WEBDAV_PASS and UPLOAD_PATH 
    #      should be retrieved by a Secret manager or at least with masked gitlab variables https://docs.gitlab.com/ee/ci/variables/#masked-variable-requirements
    only:
        refs:
        - master #Only upload job if its the master branch
    when: manual #Release is a manual job,otherwise would uplaod each new commit

