# qfileinfo-exists

Finds places using `QFileInfo("foo").exists()` instead of the faster version `QFileInfo::exists("foo")`.

According to Qt's docs:
"Using this function is faster than using QFileInfo(file).exists() for file system access."
