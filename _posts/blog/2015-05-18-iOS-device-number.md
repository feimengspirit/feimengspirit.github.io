---
layout: post
title: iOS设备名列表(持续更新)
description: 收集了目前为止的所有iOS设备号及其对应的设备名称
category: blog
---
	
iOS中涉及到机型适配的开发时，由于系统api只会返回硬件号，必须做下到机型名的映射。下面收集了目前为止的所有iOS设备的硬件号及其映射关系。后面如果有新机器还会持续更新。
	
    struct utsname systemInfo;
    uname(&systemInfo);
    NSString *platform 	= [NSString stringWithCString:systemInfo.machine
                                             encoding:NSUTF8StringEncoding];

	if ([platform isEqualToString:@"iPhone1,1"]) return @"iPhone";
	if ([platform isEqualToString:@"iPhone1,2"]) return @"iPhone 3G";
	if ([platform isEqualToString:@"iPhone2,1"]) return @"iPhone 3GS";
	if ([platform isEqualToString:@"iPhone3,1"]) return @"iPhone 4";
	if ([platform isEqualToString:@"iPhone3,2"]) return @"iPhone 4";
	if ([platform isEqualToString:@"iPhone3,3"]) return @"iPhone 4 (Verizon)";
	if ([platform isEqualToString:@"iPhone4,1"]) return @"iPhone 4S";
	if ([platform isEqualToString:@"iPhone5,1"]) return @"iPhone 5 (GSM)";
	if ([platform isEqualToString:@"iPhone5,2"]) return @"iPhone 5 (GSM+CDMA)";
	if ([platform isEqualToString:@"iPhone5,3"]) return @"iPhone 5c (GSM+CDMA)";
	if ([platform isEqualToString:@"iPhone5,4"]) return @"iPhone 5c (UK+Europe+Asia+China)";
	if ([platform isEqualToString:@"iPhone6,1"]) return @"iPhone 5s (GSM+CDMA)";
	if ([platform isEqualToString:@"iPhone6,2"]) return @"iPhone 5s (UK+Europe+Asia+China)";
	if ([platform isEqualToString:@"iPhone7,1"]) return @"iPhone 6 Plus";
	if ([platform isEqualToString:@"iPhone7,2"]) return @"iPhone 6";
	
	if ([platform isEqualToString:@"iPod1,1"])   return @"iPod Touch 1G";
	if ([platform isEqualToString:@"iPod2,1"])   return @"iPod Touch 2G";
	if ([platform isEqualToString:@"iPod3,1"])   return @"iPod Touch 3G";
	if ([platform isEqualToString:@"iPod4,1"])   return @"iPod Touch 4G";
	if ([platform isEqualToString:@"iPod5,1"])   return @"iPod Touch 5G";
	
	if ([platform isEqualToString:@"iPad1,1"])   return @"iPad";
	if ([platform isEqualToString:@"iPad2,1"])   return @"iPad 2 (WiFi)";
	if ([platform isEqualToString:@"iPad2,2"])   return @"iPad 2 (GSM)";
	if ([platform isEqualToString:@"iPad2,3"])   return @"iPad 2 (CDMA)";
	if ([platform isEqualToString:@"iPad2,4"])   return @"iPad 2";
	if ([platform isEqualToString:@"iPad2,5"])   return @"iPad mini-1G (WiFi)";
	if ([platform isEqualToString:@"iPad2,6"])   return @"iPad mini-1G (GSM)";
	if ([platform isEqualToString:@"iPad2,7"])   return @"iPad mini-1G (GSM+CDMA)";
	if ([platform isEqualToString:@"iPad3,1"])   return @"iPad-3G (WiFi)";
	if ([platform isEqualToString:@"iPad3,2"])   return @"iPad-3G (4G)";
	if ([platform isEqualToString:@"iPad3,3"])   return @"iPad-3G (4G)";
	if ([platform isEqualToString:@"iPad3,4"])   return @"iPad-4G (WiFi)";
	if ([platform isEqualToString:@"iPad3,5"])   return @"iPad-4G (GSM)";
	if ([platform isEqualToString:@"iPad3,6"])   return @"iPad-4G (GSM+CDMA)";
	if ([platform isEqualToString:@"iPad4,1"])   return @"iPad Air (WiFi)";
	if ([platform isEqualToString:@"iPad4,2"])   return @"iPad Air (GSM+CDMA)";
	if ([platform isEqualToString:@"iPad4,3"])   return @"iPad Air (GSM+CDMA)";
	if ([platform isEqualToString:@"iPad4,4"])   return @"iPad Mini Retina (WiFi)";
	if ([platform isEqualToString:@"iPad4,5"])   return @"iPad Mini Retina (GSM+CDMA)";
	if ([platform isEqualToString:@"iPad5,3"])   return @"iPad Air 2 (WiFi)";
	if ([platform isEqualToString:@"iPad5,4"])   return @"iPad Air 2";
	
	if ([platform isEqualToString:@"i386"])      return @"Simulator";
	if ([platform isEqualToString:@"x86_64"])    return @"Simulator";
