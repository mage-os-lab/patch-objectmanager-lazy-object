# PHP 8.4 Lazy Object in ObjectManager patch

**This repository contains a patch for Magento Open Source and Mage-OS
to allow the ObjectManager to use PHP 8.4 Lazy Objects.**

## Status
Experimental

## Background
PHP 8.4 introduced the concept of lazy objects. This PHP-native mechanism somewhat resembles the Magento 2 DI proxy pattern. However, when
implemented properly, the performance of Magento with lazy objects could potentially shave off dozens of milliseconds off every request. 

Benjamin Eberlei (Tideways) created the following Youtube videos to illustrate this: https://www.youtube.com/watch?v=qmzoys3WAXE&t=267s

This repository contains a patched based on the patch Benjamin describes in his video.

## Usage

To apply this patch to your Mage-OS installation use [cweagans/composer-patches](https://github.com/cweagans/composer-patches).

Once you've set this up follow these steps:

1. Download the patch in this repository and save it to your Mage-OS root. Eg. patches/lazyObjectsPatch.patch
2. Configure your composer.json to install the patch on mage-os/framework:

```json
  "extra": {
    "magento-force": "override",
      "composer-exit-on-patch-failure": true,
      "patches": {
          "mage-os/framework": {
              "Use PHP 8.4 Native Lazy Loading": "./patches/lazyObjectsPatch.patch"
          }
      }
  },
```

3. Install the patches as described in the composer patches documentation: https://docs.cweagans.net/composer-patches/usage/recommended-workflows/

## Contributing

Right now this patch is in an experimental state and hasn't been battle-tested on production systems. If you want to work on this patch there are a few important things to know:

### Limitations of Native Lazy Objects

There are a few limitations of Native Lazy Objects that you should know about. 

- Native Lazy Objects are not supporter in PHP versions below 8.4
- You cannot create a Lazy Proxy of an internal PHP class other than stdClass.

As of right now we've prevented errors related to this with the following checks:

```php
    \PHP_VERSION_ID >= 80400 && // Only PHP 8.4 or higher
    !\is_subclass_of($finalClass, \NumberFormatter::class) && // Not a subclass of the NumberFormatter internal class
    !\is_subclass_of($finalClass, \SessionHandler::class) // Not a subclass of the SessionHandler internal class
```

There exists an alternative version of this patch that has more robust checking for Interal classes at the cost of a small performance penalty. It looks something like this:

```php
    $finalClass = $this->config->getInstanceType($requestedType);

    $reflectionClass = new \ReflectionClass($finalClass);
    $currentClass = $reflectionClass;

    do {
        if ($currentClass->isInternal()) {
            return $this->doCreate($requestedType, $arguments);
        }
    } while ($currentClass = $currentClass->getParentClass());

    if (\PHP_VERSION_ID >= 80400 &&
        !\str_ends_with($finalClass, '\\Proxy') &&
        $finalClass !== \Magento\Framework\View\Page\Builder::class &&
        !\is_subclass_of($finalClass, \Magento\Framework\Session\SessionManager::class)
    ) {
        return $reflectionClass->newLazyProxy(
            function ($object) use ($requestedType, $arguments) {
                return $this->doCreate($requestedType, $arguments);
            }
        );
    }

    return $this->doCreate($requestedType, $arguments);
```

### Magento Limitations

There are a few Magento specific limitations we've experienced as well.

- Making a Lazy Proxy of the \Mageno\Framework\View\Page\Builder class breaks output rendering
- There is no performance benefit to creating a Lazy Proxy of a Proxy class
- Making a Lazy Proxy of Session related classes causes the session_start function to be called multiple times

To deal with these limitations we've added the follwing checks:

```php
    !\str_ends_with($finalClass, '\\Proxy') && // Do not proxy Proxy classes
    $finalClass !== \Magento\Framework\View\Page\Builder::class && // Do not proxy the Builder class
    !\is_subclass_of($finalClass, \Magento\Framework\Session\SessionManager::class) // Do not proxy subclasses of the SessionManager
```
