# Xamarin.ZoomSDKBinding
 
* Android: Meeting SDK Version: 5.12.2.9109
 
* iOS: 
 	* MobileRTC Version: 5.11.10.4556
 	* MobileRTC Screen Share Version: Not implemented yet, PRs are welcome! 
 
## Android Gotchas
* Requires your android app to compile for Android 12

* The csproj of your android app needs a particular version of AndroidX.Core which is out of the bounds of the latest Xamarin.Forms version. It is likely to remain so with MAUI here now, and XF development at a minimum from Microsoft. I haven't noticed any negative consequences in my XF app, but it is something to watch out for in your implementation.

```
<PackageReference Include="Xamarin.AndroidX.Core">
      <Version>1.8.0.1</Version>
</PackageReference>
```

## Installation and integration - Android
 
1. Grab the package from nuget ```Install-Package VisualService.Xamarin.Android.ZoomSDK``` and install it as a dependency to your Xamarin.Android platform project. [Package link](https://www.nuget.org/packages/VisualService.Xamarin.Android.ZoomSDK/).

2. Implement a IZoomSDKInitializeListener singleton somewhere in your application

3. Initialise the SDK in the IZoomSDKInitializeListener. Note the main thread access in this step and all the way down. It's needed.

```
            zoomSDK = ZoomSDK.Instance;
            var zoomInitParams = new ZoomSDKInitParams
            {
                AppKey = appKey,
                AppSecret = appSecret,
            };
            zoomSDK.Initialize(Android.App.Application.Context, this, zoomInitParams);
```

4. Listen for successful initialisation inside IZoomSDKInitializeListener.OnZoomSDKInitializeResult

```
        public void OnZoomSDKInitializeResult(int errorCode, int internalErrorCode)
        {
            if (errorCode == ZoomError.ZoomErrorSuccess)
            {
                //Add listeners according to your needs
                //zoomSDK.InMeetingService.AddListener(new YourInMeetingServiceListener());
                //zoomSDK.MeetingService.AddListener(new YourMeetingServiceListener());
            }
            else
            {
                // something bad happened
            }
        }
```

6. Join Meeting

```
            var meetingService = zoomSDK.MeetingService;
            var resultJoinMeeting = meetingService.JoinMeetingWithParams(Android.App.Application.Context, new JoinMeetingParams
            {
                MeetingNo = meetingID,
                DisplayName = displayName,
                Password = meetingPassword
            }, new JoinMeetingOptions { });
 ```


## Installation and integration - iOS

1. Grab the package from nuget ```Install-Package VisualService.Xamarin.iOS.ZoomSDK``` and install it as a dependency to your Xamarin.iOS platform project.
 
2. Implement an IMobileRTCMeetingServiceDelegate (which should also inherit from your "Cross platform interface" as a DependencyService);
 
3. In the IMobileRTCMeetingServiceDelegate class you should initialize the SDK as follows: 
 
 ```
        public void InitZoomLib(string appKey, string appSecret)
        {
            bool InitResult;

            Device.BeginInvokeOnMainThread(() => { 
                InitResult = mobileRTC.Initialize(new MobileRTCSDKInitContext
                {
                    EnableLog = true,
                    Domain = "https://zoom.us",
                    Locale = MobileRTC_ZoomLocale.Default
                });
                if (InitResult)
                {
                    mobileRTC.SetLanguage("it");
                    authService = mobileRTC.GetAuthService();
                    if (authService != null)
                    {
                        authService.Delegate = new MobileDelegate();   //inherits from MobileRTCAuthDelegate
                        authService.ClientKey = appKey;
                        authService.ClientSecret = appSecret;
                        authService.SdkAuth();
                    }
                    Console.WriteLine($"Mobile RTC Version: {mobileRTC.MobileRTCVersion()} ");
                }
            });
        } 
```

4. Listen for successful initialisation inside MobileRTCAuthDelegate
```
        class MobileDelegate : MobileRTCAuthDelegate
        {
            public override void OnMobileRTCAuthReturn(MobileRTCAuthError returnValue)
            {
                Console.WriteLine($"Another Log from our iOS counterpart: {returnValue}");
            }
        }
```

4.1 You can monitor the initialisation and authorization status in any moment like this: 

```
        public bool IsInitialized()
        {
            bool result = false;

            if (MobileRTC.SharedRTC != null)
            {
                if (MobileRTC.SharedRTC.IsRTCAuthorized())
                {
                    if (MobileRTC.SharedRTC.GetMeetingService() != null)
                    {
                        result = true;
                    }
                }
            }
            return result;
        }
```

5. Join the meeting 

```
     public void JoinMeeting(string meetingID, string meetingPassword, string displayName = "Zoom Demo")
     {
            if (IsInitialized())
            {
                var meetingService = mobileRTC.GetMeetingService();
                meetingService.Init();
                meetingService.Delegate = this;

                MobileRTCMeetingJoinParam meetingParamDict = new MobileRTCMeetingJoinParam();
                meetingParamDict.UserName = displayName;
                meetingParamDict.MeetingNumber = meetingID;
                meetingParamDict.Password = meetingPassword;

                MobileRTCMeetingSettings settings = mobileRTC.GetMeetingSettings();
                settings.DisableDriveMode(true);
                settings.EnableCustomMeeting = false;
                //Specify your meeting options here

                Device.BeginInvokeOnMainThread(() =>
                {
                    var meetingJoinResponse = meetingService.JoinMeetingWithJoinParam(meetingParamDict);
                    Console.WriteLine($"Meeting Joining Response {meetingJoinResponse}");
                });
            }
	}
```

6. Monitor the meeting status
```
        public override void OnMeetingStateChange(MobileRTCMeetingState state)
        {
            if (state == MobileRTCMeetingState.Ended)
            {
                //The meeting has ended
                //=> Fire your event! 
            }
            if (state == MobileRTCMeetingState.InMeeting)
            {
                inMeeting = true;
            }
            else
            {
                inMeeting = false;
            }
        }
```

## Contributing

You are welcome to raise issues. PRs are particularly welcome, as the maintainer's primary focus is a commercial product which only uses certain limited feature of the Zoom SDK. Therefore time to spend on fixing issues not directly related to features we require will be limited.

## Building locally - Android

### Format correctly the .aar file (Unneccessary in 5.12.2.9109 - but be aware zoom may break things in future versions)

If you download a fresh android sdk .aar file to upgrade the version, before it will build, you will need to run a manual process to strip out incorrectly formatted placeholder characters present in the zoom sdk resource files. Basically %s and %d characters need replacing with their positional alternatives %1$s and %2$d etc. (I don't know how it compiles for the zoom guys themselves without this, but perhaps the native android build process handles this kind of thing for you?) 

There are hundreds of resource files, so I am including a replace utility console app in the src. Instructions for use are below. A PR to automate this process further is welcome. I also raised [an issue in the zoom developer forums](https://devforum.zoom.us/t/2-multiple-substitutions-still-specified-in-non-positional-format/63243) to fix this at their end, but there is no sign of a fix yet.

1. Download the latest zoom sdk
2. Inside the mobile RTC folder, find the file called mobilertc.aar and rename it to mobilertc.zip
3. Extract the contents of the folder.
4. Run the replace utility console app, located in src/Droid/, making sure to point the file location in the script to the res folder inside the extracted folder
5. Recompile the mobilertc.aar file with this command ```jar cvf mobilertc.aar -C theExtractedFolderName/ .```
6. Your mobilertc.aar file will now be suitable to use in the binding project.

### To create the nuget package

1. Change the Version node in the file MobileRTC_Droid.csproj to the latest version
2. Build in release mode
3. The nuget package will appear in the bin/Release folder

## History

This project is originally based on [this repo](https://github.com/stntz/Xamarin.ZoomBinding)
