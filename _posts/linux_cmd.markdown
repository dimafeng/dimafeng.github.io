mysqldump -u [uname] -p[pass] [dbname] | gzip -9 > [backupfile.sql.gz]
gunzip < alldb.sql.gz | mysql -u root -ppassword db

"also, if you want to cycle through the different commands that contain the string you just typed, keep on pressing ctrl+r"


tar -zxvf file.tar.gz
-z : Work on gzip compression automatically when reading archives.
-x : Extract archives.
-v : Produce verbose output i.e. display progress and extracted file list on screen.
-f : Read the archive from the archive to the specified file. In this example, read backups.tar.gz archive.
-t : List the files in the archive.

tar -cvzf test.tgz ./*
c --- to create a tar file, writing the file starts at the beginning.
t --- table of contents, see the names of all files or those specified in other command line arguments.
x --- extract (restore) the contents of the tar file.
f --- specifies the filename (which follows the f) used to tar into or to tar out from; see the examples below.
z --- use zip/gzip to compress the tar file or to read from a compressed tar file.
v --- verbose output, show, e.g., during create or extract, the files being stored into or restored from the tar file.


http://files.fosswire.com/2007/08/fwunixref.pdf
