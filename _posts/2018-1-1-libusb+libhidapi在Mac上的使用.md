---
layout: post
title:
modified:
categories: Tech

tags: [linux, libusb]

comments: true
---

<!-- TOC -->

- [起因](#起因)
- [解决](#解决)

<!-- /TOC -->

### 起因

项目的一个小工具，将鼠标控制 forward 到 Android 平台上，通过 pc 中转，最初用的 Ubuntu,核心自然是 libusb 的使用了，一切倒还顺利，基本的思路是:

> switch android aoa mode
>
> libusb access hid mouse device, get hid report descriptor.
>
> send descriptor to android.
>
> forward hid report data to android;

到了 MAC 电脑上，发现用不了了。检查了半天，发现是在 open hid device 后的
`claim_interface`失败.

```c
    printf("libusb_kernel_driver_active %d\n",interface);
	ret = libusb_kernel_driver_active(*handle, interface);
	if (ret == 1) {
		if (libusb_detach_kernel_driver(*handle, interface)) {
			printf("Unable to grab usb device\n");
			libusb_close(*handle);
			*handle = NULL;
			return -1;
		}
		kernel_claimed = 1;
	}
    printf("libusb_claim_interface\n");
	ret = libusb_claim_interface(*handle, interface);
	if (ret) {
		printf("Failed to claim interface %d.\n", interface);
```

google 了很久，终于找到[libusb 的 issue 讨论](https://github.com/libusb/libusb/issues/158),有人提到 Mac 上做 hid 设备时，最好不要用 libusb,而是 libusbapi,这是个无驱方案，也就是不需要上面的`detach_kernel`动作，而且加了这个反倒不正常了.

[libhidapi 在这里](http://www.signal11.us/oss/hidapi/)

但是！hidapi 并没有办法 get hid descriptor,只有 libusb 可以做到.

怎么办呢，只好结合两者了

### 解决

原来 libusb 的逻辑不动，把 libusb asynchronous 中的 interput transfer 接收 hid report 的方式改为用 hidapi 的`hid_read`(的确简单了）但是 hid_read 是不能在 kernel detach 的方式下用的。重点来了,再 send descriptor 到 android 后，因为 hid 的 handle 后面用 hidapi 代替操作了，不再需要用了.可以通过下面的方法恢复 driver:

```c
// 0 is interface number
libusb_release_interface(hid->handle, 0);
libusb_attach_kernel_driver(hid->handle,0);
libusb_close(hid->handle);
```

在这之后，再尽情使用 hidapi 好了:

```
void * rx_hidapi(void* para)
{
	accessory_t * acc = (accessory_t *)para;
	unsigned char buf[256] = {0};
	hid_device *handle;
	struct hid_device_info *devs, *cur_dev;
	printf("rx_hidapi start..\n");
	if (hid_init())
		return -1;
	devs = hid_enumerate(0x0, 0x0);
	printf("hid_enumerate\n");
	cur_dev = devs;
	int pid,vid;
	wchar_t* found = NULL;
	while (cur_dev) {
		printf("Device Found\n  type: %04hx %04hx\n  path: %s\n  serial_number: %ls", cur_dev->vendor_id, cur_dev->product_id, cur_dev->path, cur_dev->serial_number);
		printf("\n");
		printf("  Manufacturer: %ls\n", cur_dev->manufacturer_string);
		printf("  Product:      %ls\n", cur_dev->product_string);
		printf("  Release:      %hx\n", cur_dev->release_number);
		printf("  Interface:    %d\n",  cur_dev->interface_number);
		printf("\n");
		// wchar_t using wcsstr replace strstr.
		found = wcsstr(cur_dev->product_string,L"Mouse");
		if (found) {
			printf("Mouse found\n");
			pid = cur_dev->product_id;
			vid = cur_dev->vendor_id;
			break;
		} else {
			printf("Looking up Mouse ...\n");
			cur_dev = cur_dev->next;
		}
	}
	hid_free_enumeration(devs);

	printf("hid_open\n");
	handle = hid_open(vid, pid, NULL);
	/*handle = hid_open(0x413c, 0x301a, NULL);*/
	if (!handle) {
		printf("unable to open device\n");
 		return 1;
	}
	int res = -1;
	while(1) {
		//block reading
		res = hid_read(handle, buf, sizeof(buf));
		if (res > 0) {
			callback_hidapi(acc , buf, res);
		} else {
			printf("error during hid_read %d\n",res);
			return res;
		}
	}
	hid_exit();
}
```

这个方案在 linux,mac 和 windows 应该都是通用的.

推到 github 上了 [here](https://github.com/anribras/linux-adk)
