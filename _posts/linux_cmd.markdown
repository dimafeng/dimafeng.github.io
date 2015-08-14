mysqldump -u [uname] -p[pass] [dbname] | gzip -9 > [backupfile.sql.gz]


"also, if you want to cycle through the different commands that contain the string you just typed, keep on pressing ctrl+r"


tar -zxvf file.tar.gz
-z : Work on gzip compression automatically when reading archives.
-x : Extract archives.
-v : Produce verbose output i.e. display progress and extracted file list on screen.
-f : Read the archive from the archive to the specified file. In this example, read backups.tar.gz archive.
-t : List the files in the archive.
