```dockerfile
# Setup binary path
ENV PATH=${PATH}:${ONESPINROOT}/bin

# Configure runtime or shared libraries search path, appending existing LD_LIBRARY_PATH when available.
ENV LD_LIBRARY_PATH="${ONESPINROOT}/lib/Linux_x86_64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}"
```