# 最简单的DRM应用程序

标签（空格分隔）： 未分类

---
最近在学习DRM (Direct Rendering Manager)驱动程序，在这里将学习的经验总结分享给大家。

在学习DRM驱动之前，应该首先了解如何使用DRM驱动。以下使用伪代码的方式，简单介绍如何编写一个最简单的DRM应用程序。
```c
int main(int argc, char **argv)
{
    /* open the drm device */
	open("/dev/dri/card0");

    /* get crtc/encoder/connector id */
	drmModeGetResources(...);

    /* get connector for display mode */
	drmModeGetConnector(...);

    /* create a dumb-buffer */
	drmIoctl(DRM_IOCTL_MODE_CREATE_DUMB);

    /* bind the dumb-buffer to a FB object */
	drmModeAddFB(...);

    /* map the dumb buffer for drawing */
	drmIoctl(DRM_IOCTL_MODE_MAP_DUMB);
	mmap(...);

    /* start display */
    drmModeSetCrtc(crtc_id, fb_id, connector_id, mode);
}
```

当执行完`mmap`之后，我们就可以直接在应用层对framebuffer进行操作了。

详细参考代码如下：
```c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <time.h>
#include <unistd.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

struct buffer_object {
	uint32_t width;
	uint32_t height;
	uint32_t pitch;
	uint32_t handle;
	uint32_t size;
	uint8_t *vaddr;
	uint32_t fb_id;
};

struct buffer_object buf;

static int modeset_create_fb(int fd, struct buffer_object *bo)
{
	struct drm_mode_create_dumb create = {};
 	struct drm_mode_map_dumb map = {};

	create.width = bo->width;
	create.height = bo->height;
	create.bpp = 32;
	drmIoctl(fd, DRM_IOCTL_MODE_CREATE_DUMB, &create);

	bo->pitch = create.pitch;
	bo->size = create.size;
	bo->handle = create.handle;
	drmModeAddFB(fd, bo->width, bo->height, 24, 32, bo->pitch,
			   bo->handle, &bo->fb_id);

	map.handle = create.handle;
	drmIoctl(fd, DRM_IOCTL_MODE_MAP_DUMB, &map);

	bo->vaddr = mmap(0, create.size, PROT_READ | PROT_WRITE,
			MAP_SHARED, fd, map.offset);

	memset(bo->vaddr, 0xff, bo->size);

	return 0;
}

static void modeset_destroy_fb(int fd, struct buffer_object *bo)
{
	struct drm_mode_destroy_dumb destroy = {};

	drmModeRmFB(fd, bo->fb_id);

	munmap(bo->vaddr, bo->size);

	destroy.handle = bo->handle;
	drmIoctl(fd, DRM_IOCTL_MODE_DESTROY_DUMB, &destroy);
}

int main(int argc, char **argv)
{
	int fd;
	drmModeConnector *conn;
	drmModeRes *res;
	uint32_t conn_id;
	uint32_t crtc_id;

	fd = open("/dev/dri/card0", O_RDWR | O_CLOEXEC);

	res = drmModeGetResources(fd);
	crtc_id = res->crtcs[0];
	conn_id = res->connectors[0];

	conn = drmModeGetConnector(fd, conn_id);
	buf.width = conn->modes[0].hdisplay;
	buf.height = conn->modes[0].vdisplay;

	modeset_create_fb(fd, &buf);

	drmModeSetCrtc(fd, crtc_id, buf.fb_id,
			0, 0, &conn_id, 1, &conn->modes[0]);

	getchar();

	modeset_destroy_fb(fd, &buf);

	drmModeFreeConnector(conn);
	drmModeFreeResources(res);

	close(fd);

	return 0;
}
```
需要注意的是，以上参考代码没有做任何函数返回值的判断，且只有在如下条件都满足的情况下，才能正常运行：
> 1. DRM驱动支持MODESET；
2. DRM驱动支持dumb-buffer(即连续物理内存)；
3. DRM驱动至少支持1个CRTC，1个Encoder，1个Connector；
4. DRM驱动的Connector至少支持1个drm_display_mode。

源码连接：https//xxxx

参考资料：
David Herrmann's Github: [drm-howto/modeset.c][1] 

[1]: https://github.com/dvdhrm/docs/blob/master/drm-howto/modeset.c