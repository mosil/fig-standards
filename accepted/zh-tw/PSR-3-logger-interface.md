記錄器介面
==========

這份文件 (PSR-3) 說明一項記錄用函式庫所該具有的共通介面。

文件的主要目的在於提供簡潔並一致的方式，讓記錄函式庫能和依照 `Psr\Log\LoggerInterface` 介面實做的物件溝通並寫入記錄。同樣的，框架和內容管理系統可以 (MAY) 依各自的需求客製並延伸、擴充此介面，但需要 (SHOULD) 保留和本文件的相容性。此介面規範能確保程式所使用的第三方套件能將記錄回寫至主應用程式。

在文件中所使用到的關鍵字 “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, 以及 “OPTIONAL” 皆引用自 [RFC 2199][] 中說明。

此外，`實做者` (implementor) 一詞在文件中代表實做自 `LoggerInterface` 並和記錄相關的函式庫或框架，`使用者` (user) 則代表記錄器的操作者。

[RFC 2119]: http://tools.ietf.org/html/rfc2119

1. 實做規範
-----------

### 1.1 基礎項目

- `LoggerInterface` 需提供八個紀錄寫入函式，對應至八種 [RFC 5424][] 紀錄層級
  (debug, info, notice, warning, error, critical, alert, emergency)。

- 第九個函式，`log`，必須以紀錄層級 (log level) 作為第一個傳入參數，
  並且在呼叫時一定 (MUST) 會得到和該紀錄層級自身函式相同的執行結果。
  若呼叫此函式時傳遞了一個不在此份規範內，且實做項目 (implementation) 無法辨識的紀錄層級，
  此函式一定 (MUST) 會拋出 `Psr\Log\InvalidArgumentException`。
  使用者不應當 (SHOULD NOT) 使用任何實做項目 (implementation) 已知必定支援之外的自定義紀錄層級。

[RFC 5424]: http://tools.ietf.org/html/rfc5424

### 1.2 Message

- Every method accepts a string as the message, or an object with a
  `__toString()` method. Implementors MAY have special handling for the passed
  objects. If that is not the case, implementors MUST cast it to a string.

- The message MAY contain placeholders which implementors MAY replace with
  values from the context array.

  Placeholder names MUST correspond to keys in the context array.

  Placeholder names MUST be delimited with a single opening brace `{` and
  a single closing brace `}`. There MUST NOT be any whitespace between the
  delimiters and the placeholder name.

  Placeholder names SHOULD be composed only of the characters `A-Z`, `a-z`,
  `0-9`, underscore `_`, and period `.`. The use of other characters is
  reserved for future modifications of the placeholders specification.

  Implementors MAY use placeholders to implement various escaping strategies
  and translate logs for display. Users SHOULD NOT pre-escape placeholder
  values since they can not know in which context the data will be displayed.

  The following is an example implementation of placeholder interpolation
  provided for reference purposes only:

  ```php
  /**
   * Interpolates context values into the message placeholders.
   */
  function interpolate($message, array $context = array())
  {
      // build a replacement array with braces around the context keys
      $replace = array();
      foreach ($context as $key => $val) {
          $replace['{' . $key . '}'] = $val;
      }

      // interpolate replacement values into the message and return
      return strtr($message, $replace);
  }

  // a message with brace-delimited placeholder names
  $message = "User {username} created";

  // a context array of placeholder names => replacement values
  $context = array('username' => 'bolivar');

  // echoes "Username bolivar created"
  echo interpolate($message, $context);
  ```

### 1.3 Context

- Every method accepts an array as context data. This is meant to hold any
  extraneous information that does not fit well in a string. The array can
  contain anything. Implementors MUST ensure they treat context data with
  as much lenience as possible. A given value in the context MUST NOT throw
  an exception nor raise any php error, warning or notice.

- If an `Exception` object is passed in the context data, it MUST be in the
  `'exception'` key. Logging exceptions is a common pattern and this allows
  implementors to extract a stack trace from the exception when the log
  backend supports it. Implementors MUST still verify that the `'exception'`
  key is actually an `Exception` before using it as such, as it MAY contain
  anything.

### 1.4 Helper classes and interfaces

- The `Psr\Log\AbstractLogger` class lets you implement the `LoggerInterface`
  very easily by extending it and implementing the generic `log` method.
  The other eight methods are forwarding the message and context to it.

- Similarly, using the `Psr\Log\LoggerTrait` only requires you to
  implement the generic `log` method. Note that since traits can not implement
  interfaces, in this case you still have to `implement LoggerInterface`.

- The `Psr\Log\NullLogger` is provided together with the interface. It MAY be
  used by users of the interface to provide a fall-back "black hole"
  implementation if no logger is given to them. However conditional logging
  may be a better approach if context data creation is expensive.

- The `Psr\Log\LoggerAwareInterface` only contains a
  `setLogger(LoggerInterface $logger)` method and can be used by frameworks to
  auto-wire arbitrary instances with a logger.

- The `Psr\Log\LoggerAwareTrait` trait can be used to implement the equivalent
  interface easily in any class. It gives you access to `$this->logger`.

- The `Psr\Log\LogLevel` class holds constants for the eight log levels.

2. Package
----------

The interfaces and classes described as well as relevant exception classes
and a test suite to verify your implementation is provided as part of the
[psr/log](https://packagist.org/packages/psr/log) package.

3. `Psr\Log\LoggerInterface`
----------------------------

```php
<?php

namespace Psr\Log;

/**
 * Describes a logger instance
 *
 * The message MUST be a string or object implementing __toString().
 *
 * The message MAY contain placeholders in the form: {foo} where foo
 * will be replaced by the context data in key "foo".
 *
 * The context array can contain arbitrary data, the only assumption that
 * can be made by implementors is that if an Exception instance is given
 * to produce a stack trace, it MUST be in a key named "exception".
 *
 * See https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-3-logger-interface.md
 * for the full interface specification.
 */
interface LoggerInterface
{
    /**
     * System is unusable.
     *
     * @param string $message
     * @param array $context
     * @return null
     */
    public function emergency($message, array $context = array());

    /**
     * Action must be taken immediately.
     *
     * Example: Entire website down, database unavailable, etc. This should
     * trigger the SMS alerts and wake you up.
     *
     * @param string $message
     * @param array $context
     * @return null
     */
    public function alert($message, array $context = array());

    /**
     * Critical conditions.
     *
     * Example: Application component unavailable, unexpected exception.
     *
     * @param string $message
     * @param array $context
     * @return null
     */
    public function critical($message, array $context = array());

    /**
     * Runtime errors that do not require immediate action but should typically
     * be logged and monitored.
     *
     * @param string $message
     * @param array $context
     * @return null
     */
    public function error($message, array $context = array());

    /**
     * Exceptional occurrences that are not errors.
     *
     * Example: Use of deprecated APIs, poor use of an API, undesirable things
     * that are not necessarily wrong.
     *
     * @param string $message
     * @param array $context
     * @return null
     */
    public function warning($message, array $context = array());

    /**
     * Normal but significant events.
     *
     * @param string $message
     * @param array $context
     * @return null
     */
    public function notice($message, array $context = array());

    /**
     * Interesting events.
     *
     * Example: User logs in, SQL logs.
     *
     * @param string $message
     * @param array $context
     * @return null
     */
    public function info($message, array $context = array());

    /**
     * Detailed debug information.
     *
     * @param string $message
     * @param array $context
     * @return null
     */
    public function debug($message, array $context = array());

    /**
     * Logs with an arbitrary level.
     *
     * @param mixed $level
     * @param string $message
     * @param array $context
     * @return null
     */
    public function log($level, $message, array $context = array());
}
```

4. `Psr\Log\LoggerAwareInterface`
---------------------------------

```php
<?php

namespace Psr\Log;

/**
 * Describes a logger-aware instance
 */
interface LoggerAwareInterface
{
    /**
     * Sets a logger instance on the object
     *
     * @param LoggerInterface $logger
     * @return null
     */
    public function setLogger(LoggerInterface $logger);
}
```

5. `Psr\Log\LogLevel`
---------------------

```php
<?php

namespace Psr\Log;

/**
 * Describes log levels
 */
class LogLevel
{
    const EMERGENCY = 'emergency';
    const ALERT     = 'alert';
    const CRITICAL  = 'critical';
    const ERROR     = 'error';
    const WARNING   = 'warning';
    const NOTICE    = 'notice';
    const INFO      = 'info';
    const DEBUG     = 'debug';
}
```
