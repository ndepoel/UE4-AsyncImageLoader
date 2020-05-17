# Asynchronous Image Loading from Disk

Sometimes you may need to load textures from disk that are not part of your Unreal project, and thus are not packaged along with your game. Perhaps you need to load photos created by an external application, or perhaps you want to recreate the 1997 game Virus, where images found on your hard drive were used as textures in the game. The regular asset streaming systems in UE4 will only load packaged assets, so those cannot be used for this purpose.

I found myself in one such situation, so I wrote a small utility class that can load images from any location on your computer and convert them into Unreal textures at runtime. What's more, when loading images from disk you don't want the game to stutter from having to wait for the data to load, so loading is done asynchronously using a background task. Because it's fairly generic and might be useful to others, I decided to share the code for this utility on the wiki here.

The code for this class demonstrates the following:
* Loading from disk arbitrary files that were not packaged along with your game
* Converting compressed image data to a texture using the ImageWrapper module
* Creating and uploading textures at run-time
* Asynchronous programming using the Async API and Futures
* Using lambda expressions with variable capturing (i.e. closures) for callbacks
* Exposing an event to Blueprint and binding a custom event

Loaded textures are assigned an owner, which by default is the object that initiates the image load. This ties the lifecycle of each texture to the object that loaded it, meaning these textures won't get lost in the ether, taking up precious memory long after they served their use.

## Setup

To get the UImageLoader class to compile, you need to add the following line to your project's .Build.cs file:

```csharp
PrivateDependencyModuleNames.AddRange(new string[] { "ImageWrapper", "RenderCore" });
```

ImageWrapper is required for identifying image formats and decompressing them to raw pixel data. RenderCore is required for creating textures at run-time, specifically so we can link against the GPixelFormats global variable.

## The Code

Add the following code files to the Source folder of your project. You should modify the precompiled header include directive and log category to fit your project.

```cpp
#pragma once

#include "PixelFormat.h"
#include "ImageLoader.generated.h"

// Forward declarations
class UTexture2D;

/**
Utility class for asynchronously loading an image into a texture.
Allows Blueprint scripts to request asynchronous loading of an image and be notified when loading is complete.
*/
UCLASS(BlueprintType)
class UImageLoader : public UObject
{
	GENERATED_BODY()

public:
	/** 
	Loads an image file from disk into a texture on a worker thread. This will not block the calling thread. 
	@return An image loader object with an OnLoadCompleted event that users can bind to, to get notified when loading is done.
	*/
	UFUNCTION(BlueprintCallable, Category = ImageLoader, meta = (HidePin = "Outer", DefaultToSelf = "Outer"))
	static UImageLoader* LoadImageFromDiskAsync(UObject* Outer, const FString& ImagePath);

	/**
	Loads an image file from disk into a texture on a worker thread. This will not block the calling thread.
	@return A future object which will hold the image texture once loading is done.
	*/
	static TFuture<UTexture2D*> LoadImageFromDiskAsync(UObject* Outer, const FString& ImagePath, TFunction<void()> CompletionCallback);

	/**
	Loads an image file from disk into a texture. This will block the calling thread until completed.
	@return A texture created from the loaded image file.
	*/
	UFUNCTION(BlueprintCallable, Category = ImageLoader, meta = (HidePin = "Outer", DefaultToSelf = "Outer"))
	static UTexture2D* LoadImageFromDisk(UObject* Outer, const FString& ImagePath);

public:
	/**
	Declare a broadcast-style delegate type, which is used for the load completed event.
	Dynamic multicast delegates are the only type of event delegates that Blueprint scripts can bind to.
	*/
	DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnImageLoadCompleted, UTexture2D*, Texture);

	/** This accessor function allows C++ code to bind to the event. */
	FOnImageLoadCompleted& OnLoadCompleted()
	{
		return LoadCompleted;
	}

private:
	/** Helper function that initiates the loading operation and fires the event when loading is done. */
	void LoadImageAsync(UObject* Outer, const FString& ImagePath);

	/** Helper function to dynamically create a new texture from raw pixel data. */
	static UTexture2D* CreateTexture(UObject* Outer, const TArray<uint8>& PixelData, int32 InSizeX, int32 InSizeY, EPixelFormat PixelFormat = EPixelFormat::PF_B8G8R8A8, FName BaseName = NAME_None);

private:
	/**
	Holds the load completed event delegate.
	Giving Blueprint access to this private variable allows Blueprint scripts to bind to the event.
	*/
	UPROPERTY(BlueprintAssignable, Category = ImageLoader, meta = (AllowPrivateAccess = true))
	FOnImageLoadCompleted LoadCompleted;

	/** Holds the future value which represents the asynchronous loading operation. */
	TFuture<UTexture2D*> Future;
};
```

```cpp
// This includes the precompiled header. Change this to whatever is relevant for your project.
#include "TestProject.h"

#include "ImageLoader.h"
#include "ImageWrapper.h"
#include "RenderUtils.h"
#include "Engine/Texture2D.h"

// Change the UE_LOG log category name below to whichever log category you want to use.
#define UIL_LOG(Verbosity, Format, ...)		UE_LOG(LogTemp, Verbosity, Format, __VA_ARGS__)

UImageLoader* UImageLoader::LoadImageFromDiskAsync(UObject* Outer, const FString& ImagePath)
{
	// This simply creates a new ImageLoader object and starts an asynchronous load.
	UImageLoader* Loader = NewObject<UImageLoader>();
	Loader->LoadImageAsync(Outer, ImagePath);
	return Loader;
}

void UImageLoader::LoadImageAsync(UObject* Outer, const FString& ImagePath)
{
	// The asynchronous loading operation is represented by a Future, which will contain the result value once the operation is done.
	// We store the Future in this object, so we can retrieve the result value in the completion callback below.
	Future = LoadImageFromDiskAsync(Outer, ImagePath, [this]()
	{
		// This is the same Future object that we assigned above, but later in time.
		// At this point, loading is done and the Future contains a value.
		if (Future.IsValid())
		{
			// Notify listeners about the loaded texture.
			LoadCompleted.Broadcast(Future.Get());
		}
	});
}

TFuture<UTexture2D*> UImageLoader::LoadImageFromDiskAsync(UObject* Outer, const FString& ImagePath, TFunction<void()> CompletionCallback)
{
	// Run the image loading function asynchronously through a lambda expression, capturing the ImagePath string by value.
	// Run it on the thread pool, so we can load multiple images simultaneously without interrupting other tasks.
	return Async<UTexture2D*>(EAsyncExecution::ThreadPool, [=]() { return LoadImageFromDisk(Outer, ImagePath); }, CompletionCallback);
}

UTexture2D* UImageLoader::LoadImageFromDisk(UObject* Outer, const FString& ImagePath)
{
	// Check if the file exists first
	if (!FPaths::FileExists(ImagePath))
	{
		UIL_LOG(Error, TEXT("File not found: %s"), *ImagePath);
		return nullptr;
	}

	// Load the compressed byte data from the file
	TArray<uint8> FileData;
	if (!FFileHelper::LoadFileToArray(FileData, *ImagePath))
	{
		UIL_LOG(Error, TEXT("Failed to load file: %s"), *ImagePath);
		return nullptr;
	}

	// Detect the image type using the ImageWrapper module
	IImageWrapperModule& ImageWrapperModule = FModuleManager::LoadModuleChecked<IImageWrapperModule>(TEXT("ImageWrapper"));
	EImageFormat::Type ImageFormat = ImageWrapperModule.DetectImageFormat(FileData.GetData(), FileData.Num());
	if (ImageFormat == EImageFormat::Invalid)
	{
		UIL_LOG(Error, TEXT("Unrecognized image file format: %s"), *ImagePath);
		return nullptr;
	}

	// Create an image wrapper for the detected image format
	IImageWrapperPtr ImageWrapper = ImageWrapperModule.CreateImageWrapper(ImageFormat);
	if (!ImageWrapper.IsValid())
	{
		UIL_LOG(Error, TEXT("Failed to create image wrapper for file: %s"), *ImagePath);
		return nullptr;
	}

	// Decompress the image data
	const TArray<uint8>* RawData = nullptr;
	ImageWrapper->SetCompressed(FileData.GetData(), FileData.Num());
	ImageWrapper->GetRaw(ERGBFormat::BGRA, 8, RawData);
	if (RawData == nullptr)
	{
		UIL_LOG(Error, TEXT("Failed to decompress image file: %s"), *ImagePath);
		return nullptr;
	}

	// Create the texture and upload the uncompressed image data
	FString TextureBaseName = TEXT("Texture_") + FPaths::GetBaseFilename(ImagePath);
	return CreateTexture(Outer, *RawData, ImageWrapper->GetWidth(), ImageWrapper->GetHeight(), EPixelFormat::PF_B8G8R8A8, FName(*TextureBaseName));
}

UTexture2D* UImageLoader::CreateTexture(UObject* Outer, const TArray<uint8>& PixelData, int32 InSizeX, int32 InSizeY, EPixelFormat InFormat, FName BaseName)
{
	// Shamelessly copied from UTexture2D::CreateTransient with a few modifications
	if (InSizeX <= 0 || InSizeY <= 0 ||
		(InSizeX % GPixelFormats[InFormat].BlockSizeX) != 0 ||
		(InSizeY % GPixelFormats[InFormat].BlockSizeY) != 0)
	{
		UIL_LOG(Warning, TEXT("Invalid parameters specified for UImageLoader::CreateTexture()"));
		return nullptr;
	}

	// Most important difference with UTexture2D::CreateTransient: we provide the new texture with a name and an owner
	FName TextureName = MakeUniqueObjectName(Outer, UTexture2D::StaticClass(), BaseName);
	UTexture2D* NewTexture = NewObject<UTexture2D>(Outer, TextureName, RF_Transient);

	NewTexture->PlatformData = new FTexturePlatformData();
	NewTexture->PlatformData->SizeX = InSizeX;
	NewTexture->PlatformData->SizeY = InSizeY;
	NewTexture->PlatformData->PixelFormat = InFormat;

	// Allocate first mipmap and upload the pixel data
	int32 NumBlocksX = InSizeX / GPixelFormats[InFormat].BlockSizeX;
	int32 NumBlocksY = InSizeY / GPixelFormats[InFormat].BlockSizeY;
	FTexture2DMipMap* Mip = new(NewTexture->PlatformData->Mips) FTexture2DMipMap();
	Mip->SizeX = InSizeX;
	Mip->SizeY = InSizeY;
	Mip->BulkData.Lock(LOCK_READ_WRITE);
	void* TextureData = Mip->BulkData.Realloc(NumBlocksX * NumBlocksY * GPixelFormats[InFormat].BlockBytes);
	FMemory::Memcpy(TextureData, PixelData.GetData(), PixelData.Num());
	Mip->BulkData.Unlock();

	NewTexture->UpdateResource();
	return NewTexture;
}
```

## Usage

Adding the above code to your project will add a new category called "Image Loader" to your arsenal of Blueprint actions. In here, you will find a function "Load Image from Disk", which will block the calling thread, and "Load Image from Disk Async", which does not block the calling thread.
Below you see an example Blueprint graph that demonstrates the latter function:

![Blueprint example](/images/ImageLoaderBlueprint.png)

Running this script will load the default Windows wallpaper into a texture and assign it to a variable. Open this texture in the inspector and you will see it looks like this:

![Loaded texture](/images/ImageLoaderTexture.png)
