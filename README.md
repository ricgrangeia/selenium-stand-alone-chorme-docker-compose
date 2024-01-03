# Selenium Standalone Chrome Docker Compose

### This took more time than it should, so i am sharing

## History

2023-12-23 - v1.0.0 - Initial version (on Gist)
2024-01-03 - v1.1.0 - Added envoirnment variables to docker-compose file and creates env_sample file and updated 
README.md


This is a simple docker-compose file to run a Selenium Standalone Chrome Server

Sample env file
```
CONTAINER_NAME="chrome"
HOSTNAME="chrome"
PORTS_VNC="5900"
PORTS_SERVER="4444"
NETWORK_NAME="chrome-network"
NETWORK_IS_EXTERNAL=False
SE_NODE_SESSION_TIMEOUT=120
SE_NODE_MAX_SESSIONS=5
```


In the PHP ho to start RemoteWebDriver

```
public function chromeDriveStart( mixed $store = '' ): array {

	$filesystem = new Filesystem();

	$defaultPath = Yii::$app->params['DEFAULT_DOWNLOAD_PATH'];

        if ( $store )
		    $defaultPath .= '/' . $store->idStore . MathHelper::generateUniqueId();
        else
            $defaultPath .= '/' . MathHelper::generateUniqueId();

        // Create the default download path if it doesn't exist
		if ( !$filesystem->exists( $defaultPath ) ) {
			$filesystem->mkdir( $defaultPath, 0774 );
		}

		$options = new ChromeOptions();
        $options->addArguments([
            '--start-maximized',  // You can add more Chrome options as needed
            '--no-sandbox',
            '--disable-dev-shm-usage',
            '--disable-gpu',
            '--safebrowsing-enable',
            '--download.directory_upgrade=true',
            '--download.prompt_for_download=false',

        ]);
        // Set desired capabilities
        $capabilities = DesiredCapabilities::chrome();
        $capabilities->setCapability(WebDriverCapabilityType::ACCEPT_SSL_CERTS, true);
        $capabilities->setCapability('se:downloadsEnabled', true);
        $capabilities->setCapability(ChromeOptions::CAPABILITY, $options);

		$driver = RemoteWebDriver::create( 'http://chrome:4444', $capabilities );

		return [ $filesystem, $defaultPath, $driver ];
	}
```
I hade nade two functions
```
 public static function seleniumGridDownloadFiles(RemoteWebDriver $driver, string $default_path): void
    {
        //Get files names from selenium grid
        $files = $driver->executeCustomCommand('/session/:sessionId/se/files');

        // For multiple files if needed
        foreach ($files['names'] as $file) {

            // Set file to download
            $file_to_download = [
                'name' => $file,
            ];
	    
            // Get file content from selenium grid to local
            $file_content = $driver->executeCustomCommand('/session/:sessionId/se/files', 'POST', $file_to_download);
	    
            // Save file
            file_put_contents($default_path . "/" . $file, $file_content['contents']);
	    
            // Decode and unzip file
            self::seleniumSystemDecode64Unzip($default_path . "/" . $file);
        }
    }


    public static function seleniumSystemDecode64Unzip(string $path_filename): void
    {
        // Decode base64
        system("base64 -d " . $path_filename . " > " . $path_filename . ".decoded");
	
        // Saves decoded file to original file
        system("mv " . $path_filename . ".decoded" . " " . $path_filename);
	
        // Unzip file
        system("zcat " . $path_filename . " > " . $path_filename . ".decoded");
	
        // Saves unzipped file to original file
        system("mv " . $path_filename . ".decoded" . " " . $path_filename);

    }
   ```

And on the crawler code after made download call

```
  	echo 'Download Orders...' . PHP_EOL;
	
	// Action to download or other way to start download on Chrome Server
	$driver->findElement(WebDriverBy::xpath("(//div[@role='columnheader']//button)[1]"))->click();
	echo 'Wait for the download to complete...' . PHP_EOL;
	sleep(10);
	
	// function to bring/get file(s) to local machine
	self::seleniumGridDownloadFiles($driver, $defaultPath);
```
And it downloads the files to the local machine, to the path defined already decoded and unziped, with the same names