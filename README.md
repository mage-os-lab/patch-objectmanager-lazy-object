# PHP 8.4 Lazy Object in ObjectManager patch

**This repository contains a patch for Magento Open Source and Mage-OS
to allow the ObjectManager to use PHP 8.4 Lazy Objects.**

## Status
Experimental

## Background
PHP 8.4 introduced the concept of lazy objects. This PHP-native mechanism somewhat resembles the Magento 2 DI proxy pattern. However, when
implemented properly, the performance of Magento with lazy objects could potentially shave off dozens of milliseconds off every request. 

Benjamin Eberlei (Tideways) created the following Youtube videos to illustrate this: https://www.youtube.com/watch?v=qmzoys3WAXE&t=267s

This repository contains the patch as created by Benjamin Eberlei.

## Usage
```bash
@todo
```
