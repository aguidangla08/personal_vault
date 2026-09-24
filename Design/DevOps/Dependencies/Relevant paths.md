```dockerfile
# Setup binary path
ENV PATH=${PATH}:${ONESPINROOT}/bin

# Configure runtime or shared libraries search path, appending existing LD_LIBRARY_PATH when available.
ENV LD_LIBRARY_PATH="${ONESPINROOT}/lib/Linux_x86_64/bin${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}"
```

Bug found, using:

```dockerfile
LD_LIBRARY_PATH="${ONESPINROOT}/lib/Linux_x86_64/${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}"
```

Made the system use LD_LIBRARY_PATH not only for onespin but for other tools, which create a compatibility problem between tools, so only what is necessary has to be added to LD_LIBRARY_PATH.

Notice though that it hasn't been checked yet that all onespin tools work properlly.