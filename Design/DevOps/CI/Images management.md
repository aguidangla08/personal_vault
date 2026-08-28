I have some questions regarding the images management in gitlab CI:
- In a pipeline each job can have a different image?
	- **Yes (per pipeline and can be overrided by job)**
	- How does the loading image time works? it only loads the image once? it loads the image for every job?
		- **The image, in case the pull policy allows it, is downloaded only once**
			- **May depend on the runner cache and if the runner is reused**
		- **In the specific job, the image is pulled or reused, but the container has to be always created**
	- it is better to have the smallest image possible for every job, or one image for everything?
	- which are the trade-offs?
		- **Better Layered images (Layered images are implemented by using: From x, extended images)**
- how do I define the image to use? should I indicate a tag ?
	- **use tags, never use :latest**
	- which are the alternatives?
		- From the perspective of **referencing an image in GitLab CI**, there are only **two mechanisms**:
			1. **Tag** (`repository:tag`)
			2. **Digest** (`repository@sha256:...`)
				1. Computed after build, longer and doesn't indicate the content in a human comprehensive way
				2. immutable