+++
title = "Tutorial: Custom Thumbnails (Almost) Everywhere"
date = "2024-12-04"
author = "Laura Lewis"
cover = "posts/tutorial_thumbnailer/cover.webp"
tags = ["software-development"]
description = "A guide to creating custom thumbnails on Windows and Linux."
showFullContent = false
readingTime = true
+++

## Background / Definitions

I work with the Source engine a lot. I recently made a C++ library for working with Source engine textures, and I'm
reasonably proud of it. While I was working on it, a friend of mine said they'd like to use it to make thumbnails for
these texture files, so they'd be much easier to work with in a file browser. I said "hey that sounds really fun" and
proceeded to steal their idea and make it myself (with permission). This is how that went down.

For simplicity, I use the terms "thumbnail creator", "thumbnail generator", "thumbnail provider", and "thumbnailer"
interchangeably to mean a program or library that handles generating thumbnails.

## Thumbnailer Types

Before getting into the process of making thumbnailers, there are two things you should know. There are two main methods
of handling thumbnail generation, both with upsides and downsides.

1. Run a command (usually calling a custom program) which creates the thumbnail image on disk.
   - Pros:
     - Easy to program.
     - Can easily use shell scripts or other scripting languages to generate the thumbnail.
   - Cons:
     - Time to generate and display a thumbnail is dependent on how fast your storage medium is.
     - The thumbnail must be read by a separate process to be displayed.
2. Hook into a library which exposes a function to get the raw thumbnail data.
   - Pros:
     - More control is given to the calling process over what to do with the data. Should it be written to disk?
       Perhaps, but maybe for some file types we don't want to cache the thumbnail because it can be generated very
       quickly.
     - Time to generate and display a thumbnail is not dependent on how fast your storage medium is.
   - Cons:
     - More difficult to program. The library will likely depend on OS-specific code, and there's usually some degree of
       boilerplate that must be added.

Linux thumbnailers use the former method. KDE's thumbnail plugins, Windows thumbnail generators, and macOS thumbnail
plugins use the latter method.

### Linux (Generic)

I started here, at the easiest end of the spectrum. Doing some basic research, I found the `tumbler` service, as well as
most Linux file browsers, read thumbnailer entries stored in `$PREFIX/share/thumbnailers` (where `$PREFIX` is usually
`/usr`) to figure out how to generate thumbnails. Thumbnail entries are simple key-value files that look very similar to
desktop entries. They begin with the text `[Thumbnailer Entry]`, and support three keys.

- `TryExec`: optional, the program to check the existence of before executing the command to create the thumbnail.
- `Exec`: required, the command that will result in the thumbnail's creation if successful.
- `MimeType`: required, a semicolon-separated and terminated list of MIME types to generate thumbnails for.

The command put in for the `Exec` key supports certain substitutions.

- `%u`: the URI pointing to the input file.
- `%i`: the path to the input file.
- `%o`: the path to the output file.
- `%s`: the maximum desired size of the thumbnail for either dimension, in pixels. This will probably be a power of two
  between 128 and 4096, inclusive.
- `%%`: the escaped percent character.

For the `MimeType` field I needed to create a custom MIME type entry, since VTF files are not part of any universally
recognized standard. This would also come in handy for the KDE thumbnailer.
[The MIME type entry standard can be found here.](https://specifications.freedesktop.org/shared-mime-info-spec) Mine was
very simple and looked like this.

{{<code language="xml" title="vtf-thumbnailer.xml">}}
<?xml version="1.0" encoding="UTF-8"?>
<mime-info xmlns="http://www.freedesktop.org/standards/shared-mime-info">
    <mime-type type="image/x-vtf">
        <comment>Valve Texture Format File</comment>
        <acronym>VTF</acronym>
        <expanded-acronym>Valve Texture Format</expanded-acronym>
        <glob-deleteall/>
        <glob pattern="*.vtf"/>
        <glob pattern="*.VTF"/>
    </mime-type>
</mime-info>
{{</code>}}

Thus, my complete thumbnailer entry looked like this.

{{<code language="toml" title="vtf-thumbnailer.thumbnailer">}}
[Thumbnailer Entry]
TryExec=/opt/vtf-thumbnailer/vtf-thumbnailer
Exec=/opt/vtf-thumbnailer/vtf-thumbnailer -i %i -o %o -s %s
MimeType=image/x-vtf;
{{</code>}}

(Yes, I install my software in `/opt`. No particular reason why, it's just how I learned to do it.)

The program uses these values to create the thumbnail. Thanks to the
[FreeDesktop thumbnail specification](https://specifications.freedesktop.org/thumbnail-spec), we can assume the output
image from a given thumbnailer will be an 8-bit PNG. The specification also says you should set some extra metadata in
the PNG, but this is only required if you'd like the thumbnails to update when their original files' contents change.
These fields are listed in the aforementioned FreeDesktop specification, and are reproduced here for convenience.

| Key | Description | Useful? |
|---|---|---|
| `Thumb::URI` | The URI of the original file. | Used to find the original file. |
| `Thumb::MTime` | The modification time of the original file. | Used to invalidate cache entries when the original file's modification time exceeds the cache entry. |
| `Thumb::Size` | The size in bytes of the original file. | Can help in verifying the original file has or hasn't changed, but mostly pointless. |
| `Thumb::Mimetype` | The file MIME type. | Mostly pointless. |
| `Description` | A description of the thumbnail for accessibility purposes. | Unknown how you'd generate this automatically. |
| `Software` | The name of the thumbnailer that generated the thumbnail. | Pointless. |

I ignored this metadata step because I was using `stb_image_write` to save the PNG, and it doesn't support setting PNG
metadata fields. It didn't really impact the thumbnailer's functionality, so I didn't want to go to the extra trouble.

---

Three files are written when installing this application. No further installation steps are required beyond writing
these files, although a reboot might be necessary to get everything working.

- `/opt/vtf-thumbnailer/vtf-thumbnailer`
- `/usr/share/mime/packages/vtf-thumbnailer.xml`
- `/usr/share/thumbnailers/vtf-thumbnailer.thumbnailer`

Linux's thumbnailer system is exceedingly simple and very pleasant to work with in general, even if it is a bit slow.

### Linux (KDE)

As I finished up work on the thumbnailer, I noticed it was not working in Dolphin. I installed Thunar, observed it
working there (which was cool), and then realized Dolphin (or really KDE in general) has a completely different
framework for serving thumbnails. This is where things start to get painful.

Dolphin uses KDE-specific plugins, which in turn depend on Qt. This is the basic C++ boilerplate, extracted from the KDE
developers' sample thumbnailer plugin repository.

{{<code language="cpp" title="kde.h">}}
#include <KIO/ThumbnailCreator>
#include <QImage>

class VTFThumbCreator : public KIO::ThumbnailCreator {
public:
	using ThumbnailCreator::ThumbnailCreator;

	KIO::ThumbnailResult create(const KIO::ThumbnailRequest& request) override;
};
{{</code>}}
{{<code language="cpp" title="kde.cpp">}}
#include "kde.h"

#include <KPluginFactory>

K_PLUGIN_CLASS_WITH_JSON(VTFThumbCreator, "plugin.json")

KIO::ThumbnailResult VTFThumbCreator::create(const KIO::ThumbnailRequest& request) {
	// return KIO::ThumbnailResult::pass(image) with image being a valid QImage
	// instance on success
	return KIO::ThumbnailResult::fail();
}

#include <kde.moc>
{{</code>}}

All KDE library plugins have a `plugin.json` file which gets packed into the binary. In this case it holds
important metadata such as whether to cache generated thumbnails, MIME types to generate thumbnails for, and the plugin
name.

{{<code language="json" title="plugin.json">}}
{
    "CacheThumbnail": false,
    "KPlugin": {
        "MimeTypes": ["image/x-vtf"],
        "Name": "Valve Texture Format Files (VTF Thumbnailer)"
    },
    "MimeType": "image/x-vtf;"
}
{{</code>}}

The plugin name will be read by Dolphin and appear as the name of the entry in the previews settings. (And by the way,
new plugins must be manually enabled there. If your plugin doesn't work, make sure it's both showing up in this menu and
that it's enabled.)

Two more files are necessary for thumbnailer plugins, the XML metadata file and the desktop entry. The metadata file is
meant for something like a storefront, where it shows the author, version, description of the plugin, and so on.

{{<code language="xml" title="info.craftablescience.vtf-thumbnailer.metainfo.xml">}}
<?xml version="1.0" encoding="utf-8"?>
<component type="addon">
    <id>info.craftablescience.vtf-thumbnailer</id>
    <metadata_license>MIT</metadata_license>
    <project_license>MIT</project_license>
    <extends>org.kde.dolphin.desktop</extends>
    <extends>org.kde.konqueror.desktop</extends>
    <extends>org.kde.krusader.desktop</extends>
    <extends>org.kde.gwenview.desktop</extends>
    <name>VTF Thumbnailer</name>
    <summary>Valve Texture Format thumbnail generator</summary>
    <description>
        <p>This plugin allow KDE software to display thumbnails for Valve Texture Format files.</p>
    </description>
    <url type="homepage">https://github.com/craftablescience/vtf-thumbnailer</url>
    <url type="bugtracker">https://github.com/craftablescience/vtf-thumbnailer/issues</url>
    <project_group>craftablescience</project_group>
    <categories>
        <category>Graphics</category>
    </categories>
    <icon type="stock">application-postscript</icon>
</component>
{{</code>}}

The desktop entry is used to identify the plugin as a thumbnailer. One important thing to note is when
`ThumbnailerVersion` is incremented after an update, all cached thumbnails created with this thumbnailer will be
regenerated if caching is enabled. (Yes, most of these keys are duplicated from the plugin JSON. I don't exactly know
why this file needs to exist.)

{{<code language="toml" title="vtf-thumbnailer.desktop">}}
[Desktop Entry]
Type=Service
Name=Valve Texture Format Files (VTF Thumbnailer)
X-KDE-ServiceTypes=ThumbCreator
MimeType=image/x-vtf;
X-KDE-Library=vtf-thumbnailer
CacheThumbnail=false
ThumbnailerVersion=1
{{</code>}}

Possibly the most important file that is needed is the CMake buildscript. CMake is essentially required here, as you
need to use the KDE CMake utilities to get the correct installation paths for the various files, find/link to KDE
Framework libraries, and create the plugin target. Unfortunately it's a lot of CMake. As with pretty much everything
else, I copied most of it from other thumbnailers, making modifications where necessary to allow it to compile for
both KDE v5 and v6 depending on the given value of `QT_MAJOR_VERSION`.

{{<code language="cmake" title="CMakeLists.txt" isCollapsed="true">}}
set(QT_MIN_VERSION "5.15.2")
set(KF_MIN_VERSION "5.92.0")
set(KDE_COMPILERSETTINGS_LEVEL "5.82")
find_package(ECM ${KF_MIN_VERSION} REQUIRED NO_MODULE)
set(CMAKE_MODULE_PATH ${ECM_MODULE_PATH})

include(ECMOptionalAddSubdirectory)
include(KDEInstallDirs${QT_MAJOR_VERSION})
include(KDECMakeSettings)
include(KDECompilerSettings NO_POLICY_SCOPE)
include(FeatureSummary)
include(ECMDeprecationSettings)

find_package(Qt${QT_MAJOR_VERSION} ${QT_MIN_VERSION} CONFIG REQUIRED COMPONENTS Gui)
find_package(KF${QT_MAJOR_VERSION} ${KF_MIN_VERSION} REQUIRED COMPONENTS KIO)

# Plugin
ecm_set_disabled_deprecation_versions(QT 5.15.2 KF 5.100.0)
set(BUILD_SHARED_LIBS ON)
kcoreaddons_add_plugin(vtf-thumbnailer INSTALL_NAMESPACE "kf${QT_MAJOR_VERSION}/vtf-thumbnailer")
target_sources(vtf-thumbnailer PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/src/common.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/src/common.h"
        "${CMAKE_CURRENT_SOURCE_DIR}/src/kde.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/src/kde.h")
target_link_libraries(vtf-thumbnailer PUBLIC
        KF${QT_MAJOR_VERSION}::KIOGui
        Qt::Gui)
install(TARGETS vtf-thumbnailer
        DESTINATION ${KDE_INSTALL_PLUGINDIR})

# Desktop
if(QT_MAJOR_VERSION STREQUAL "6")
    set(KDE_INSTALL_KSERVICESDIR "${KDE_INSTALL_DATADIR}/kio")
endif()
install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/kde/vtf-thumbnailer.desktop"
        DESTINATION ${KDE_INSTALL_KSERVICESDIR})

# Metadata
install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/kde/info.craftablescience.vtf-thumbnailer.metainfo.xml"
        DESTINATION ${KDE_INSTALL_METAINFODIR})

# MIME type info
install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/linux/vtf-thumbnailer.xml"
        DESTINATION ${KDE_INSTALL_MIMEDIR}
        RENAME "vtf-thumbnailer-kde${QT_MAJOR_VERSION}.xml")
{{</code>}}

The version of Qt that you compile against matters. If you want your thumbnails to work on KDE Plasma v5, compile
against Qt v5. Similarly, for KDE Plasma v6 compile against Qt v6. Qt guarantees compatibility for basic objects like
`QImage` for the entire duration of major versions, so use earlier minor versions of Qt to maximize compatibility. It's
expected that you will ship multiple versions of your thumbnailer built for different major KDE versions (or simply
target the latest version and call it a day, but keep in mind as of 2024 Debian still hasn't updated to KDE v6). 
Unfortunately, I couldn't figure out how to compile the plugin for KDE v6 in GitHub Actions, so I only created a release
for KDE v5.

Finally, to actually compile this code you need some packages installed. On Debian-based distros the packages are called
`extra-cmake-modules` and `libkf5kio-dev`, accessible through `apt`. This took me far longer than I'd like to admit to
figure out, as most of the official thumbnailers don't specify what packages they need to build. I found a listing of
KDE's framework packages on a forum post and slowly pared it down until these two were left.

---

Five files are written when installing this plugin. For KDE v5, the paths look like this. The computer may need a reboot
after installation. I haven't yet figured out if that's completely necessary, but it doesn't hurt.

- `/usr/lib/plugins/vtf-thumbnailer.so`
- `/usr/lib/plugins/kf5/vtf-thumbnailer/vtf-thumbnailer.so` (I still have not figured out why the library gets
  installed to two separate locations!)
- `/usr/share/mime/packages/vtf-thumbnailer-kde5.xml`
- `/usr/share/metainfo/info.craftablescience.vtf-thumbnailer.metainfo.xml`
- `/usr/share/kservices5/vtf-thumbnailer.desktop`

### Windows (Vista onward)

Hell. It's hell here. After finishing the KDE thumbnail plugin it took about 18 hours more work to figure out the
Windows thumbnail provider, and that's while referencing several examples. Maybe I'm stupid, but I prefer to think
Windows is a pile of convoluted garbage that's been left in the sun too long. I made small tweaks and changes to this
code for *ages*, and the reason it wasn't working, for presumably several hours, was that I wasn't exporting all the
necessary symbols. But the thing is, some symbols here cannot be exported from C++ with a `__declspec` declaration,
because it would be redefining symbols from some random Windows header. Making a `.def` file to control symbol
visibility is *required*.

Anyway, Windows is interesting. The thumbnail provider returns an `HBITMAP`, very much like the KDE plugin's `QImage`
but infinitely worse. Essentially it's the same style of code as the KDE plugin but with worse types and infinitely
more boilerplate. The metadata files are eschewed for the registry (of course), so registering the thumbnail
provider is fairly trivial at least. This is what the code ended up looking like.

{{<code language="cpp" title="Windows Thumbnail Provider BS - Includes">}}
#include <cstddef>
#include <vector>

#define WIN32_LEAN_AND_MEAN
#include <Windows.h>
#include <initguid.h>
#include <ShlObj_core.h>
#include <Shlwapi.h>
#include <strsafe.h>
#include <thumbcache.h>
#include <wrl.h>

#define VTF_THUMBNAILER_CLSID_STR  L"{8b206795-0606-40ca-9eac-1d049c7ff3be}"
DEFINE_GUID(VTF_THUMBNAILER_CLSID, 0x8b206795, 0x0606, 0x40ca, 0x9e, 0xac, 0x1d, 0x04, 0x9c, 0x7f, 0xf3, 0xbe);

#define GLOBAL(ret) extern "C" [[maybe_unused]] ret __stdcall
{{</code>}}

To start, we set up the GUID for the thumbnail provider. I used an online generator, since it has to be unique. Next
comes the thumbnail provider class.

{{<code language="cpp" title="Windows Thumbnail Provider BS - Thumbnail Provider" isCollapsed="true">}}
class VTFThumbnailProvider : public IThumbnailProvider, public IInitializeWithStream {
public:
	VTFThumbnailProvider()
			: refCount(1)
			, stream(nullptr) {}

	~VTFThumbnailProvider() {
		if (this->stream) {
			this->stream->Release();
			this->stream = nullptr;
		}
	}

	STDMETHOD(QueryInterface)(REFIID riid, void** ppv) override {
		static const QITAB qit[] = {
			QITABENT(VTFThumbnailProvider, IThumbnailProvider),
			QITABENT(VTFThumbnailProvider, IInitializeWithStream),
			{nullptr},
		};
		return QISearch(this, qit, riid, ppv);
	}

	STDMETHOD_(ULONG, AddRef)() override {
		return InterlockedIncrement(&this->refCount);
	}

	STDMETHOD_(ULONG, Release)() override {
		const ULONG rc = InterlockedDecrement(&this->refCount);
		if (rc == 0) {
			delete this;
			return 0;
		}
		return rc;
	}

	STDMETHOD(Initialize)(IStream* stream_, DWORD) override {
		if (!stream_) {
			return E_POINTER;
		}
		this->stream = stream_;
		this->stream->AddRef();
		return S_OK;
	}

	STDMETHOD(GetThumbnail)(UINT cx, HBITMAP* phbmp, WTS_ALPHATYPE* pdwAlpha) override {
		if (!phbmp || !pdwAlpha) {
			return E_POINTER;
		}
		if (!this->stream) {
			return E_UNEXPECTED;
		}

		STATSTG stat;
		HRESULT hr = this->stream->Stat(&stat, STATFLAG_NONAME);
		if (FAILED(hr)) {
			return hr;
		}

		ULONG bytesRead = 0;
		std::vector<std::byte> data(stat.cbSize.QuadPart);
		hr = this->stream->Read(data.data(), static_cast<ULONG>(stat.cbSize.QuadPart), &bytesRead);
		if (FAILED(hr) || bytesRead != stat.cbSize.QuadPart) {
			return E_FAIL;
		}

		// The file's data has now been read to the data vector, process it
		// Return E_UNEXPECTED if the file is broken
		// Past this point data is now storing BGRA8888 (yes, BGRA8888)
		// thumbnail image data

		BITMAPINFO bmi = {};
		bmi.bmiHeader.biSize = sizeof(BITMAPINFOHEADER);
		bmi.bmiHeader.biWidth = static_cast<LONG>(width);
		bmi.bmiHeader.biHeight = -static_cast<LONG>(height);
		bmi.bmiHeader.biPlanes = 1;
		bmi.bmiHeader.biBitCount = 32;
		bmi.bmiHeader.biCompression = BI_RGB;

		void* pBits = nullptr;
		HDC hdc = GetDC(nullptr);
		HBITMAP hBitmap = CreateDIBSection(hdc, &bmi, DIB_RGB_COLORS, &pBits, nullptr, 0);
		ReleaseDC(nullptr, hdc);

		if (!hBitmap) {
			return E_OUTOFMEMORY;
		}

		std::memcpy(pBits, data.data(), data.size());

		*phbmp = hBitmap;
		*pdwAlpha = WTSAT_ARGB;

		return S_OK;
	}

private:
	ULONG refCount;
	IStream* stream;
};

GLOBAL(HRESULT) VTFThumbnailProvider_CreateInstance(REFIID riid, void** ppv) {
	auto* pNew = new(std::nothrow) VTFThumbnailProvider;
	HRESULT hr = pNew ? S_OK : E_OUTOFMEMORY;
	if (SUCCEEDED(hr)) {
		hr = pNew->QueryInterface(riid, ppv);
		pNew->Release();
	}
	return hr;
}

typedef HRESULT(*PFNCREATEINSTANCE)(REFIID, void**);
struct CLASS_OBJECT_INIT {
	const CLSID* pClsid;
	PFNCREATEINSTANCE pfnCreate;
};
const CLASS_OBJECT_INIT c_rgClassObjectInit[] = {{&VTF_THUMBNAILER_CLSID, VTFThumbnailProvider_CreateInstance}};
{{</code>}}

The boilerplate isn't nasty yet, but working with `HBITMAP` is frustrating. For some reason it wants to read the pixel
order in reverse, so `BGRA8888` gets turned into `ARGB8888`. There are a few formats the bitmap can store pixel data in,
but for the sake of simplicity I used `BGRA8888` everywhere, even for opaque textures. There's not too much interesting
going on here so let's move on to the factory.

{{<code language="cpp" title="Windows Thumbnail Provider BS - Thumbnail Provider Factory" isCollapsed="true">}}
HINSTANCE g_hInst = nullptr;
ULONG g_cRefModule = 0;

GLOBAL(BOOL) DllMain(HINSTANCE hInstance, DWORD dwReason, void*) {
	if (dwReason == DLL_PROCESS_ATTACH) {
		g_hInst = hInstance;
		DisableThreadLibraryCalls(hInstance);
	}
	return TRUE;
}

GLOBAL(void) DllAddRef() {
	InterlockedIncrement(&g_cRefModule);
}

GLOBAL(void) DllRelease() {
	InterlockedDecrement(&g_cRefModule);
}

GLOBAL(HRESULT) DllCanUnloadNow() {
	return !g_cRefModule ? S_OK : S_FALSE;
}

class VTFThumbnailProviderFactory : public IClassFactory {
public:
	static HRESULT CreateInstance(REFCLSID clsid, const CLASS_OBJECT_INIT* pClassObjectInits, size_t cClassObjectInits, REFIID riid, void** ppv) {
		*ppv = nullptr;
		HRESULT hr = CLASS_E_CLASSNOTAVAILABLE;
		for (size_t i = 0; i < cClassObjectInits; i++) {
			if (clsid == *pClassObjectInits[i].pClsid) {
				IClassFactory *pClassFactory = new(std::nothrow) VTFThumbnailProviderFactory(pClassObjectInits[i].pfnCreate);
				hr = pClassFactory ? S_OK : E_OUTOFMEMORY;
				if (SUCCEEDED(hr)) {
					hr = pClassFactory->QueryInterface(riid, ppv);
					pClassFactory->Release();
				}
				break;
			}
		}
		return hr;
	}

	explicit VTFThumbnailProviderFactory(PFNCREATEINSTANCE pfnCreate_)
			: refCount(1)
			, pfnCreate(pfnCreate_) {
		::DllAddRef();
	}

	IFACEMETHODIMP QueryInterface(REFIID riid, void** ppv) override {
		static const QITAB qit[] = {
			QITABENT(VTFThumbnailProviderFactory, IClassFactory),
			{nullptr},
		};
		return QISearch(this, qit, riid, ppv);
	}

	IFACEMETHODIMP_(ULONG) AddRef() override {
		return InterlockedIncrement(&this->refCount);
	}

	IFACEMETHODIMP_(ULONG) Release() override {
		const ULONG rc = InterlockedDecrement(&this->refCount);
		if (!rc) {
			delete this;
		}
		return rc;
	}

	IFACEMETHODIMP CreateInstance(IUnknown* pUnkOuter, REFIID riid, void** ppv) override {
		return pUnkOuter ? CLASS_E_NOAGGREGATION : this->pfnCreate(riid, ppv);
	}

	IFACEMETHODIMP LockServer(BOOL fLock) override {
		if (fLock) {
			::DllAddRef();
		} else {
			::DllRelease();
		}
		return S_OK;
	}

private:
	~VTFThumbnailProviderFactory() {
		::DllRelease();
	}

	ULONG refCount;
	PFNCREATEINSTANCE pfnCreate;
};

GLOBAL(HRESULT) DllGetClassObject(REFCLSID clsid, REFIID riid, void** ppv) {
	return VTFThumbnailProviderFactory::CreateInstance(clsid, c_rgClassObjectInit, ARRAYSIZE(c_rgClassObjectInit), riid, ppv);
}
{{</code>}}

All of that code is boilerplate to create an instance of the thumbnail provider. The following code is more interesting.

{{<code language="cpp" title="Windows Thumbnail Provider BS - Thumbnail Provider Registry" isCollapsed="true">}}
namespace {

HRESULT SetHKCRRegistryKey(LPCWSTR subKey, LPCWSTR valueName, LPCWSTR data) {
	HKEY hKey;
	LONG result = RegCreateKeyExW(HKEY_CLASSES_ROOT, subKey, 0, nullptr, REG_OPTION_NON_VOLATILE, KEY_WRITE, nullptr, &hKey, nullptr);
	if (result != ERROR_SUCCESS) {
		return HRESULT_FROM_WIN32(result);
	}
	result = RegSetValueExW(hKey, valueName, 0, REG_SZ, (const BYTE*) data, (lstrlenW(data) + 1) * sizeof(WCHAR));
	RegCloseKey(hKey);
	return HRESULT_FROM_WIN32(result);
}

HRESULT DeleteHKCRRegistryKey(LPCWSTR subKey) {
	LONG result = SHDeleteKeyW(HKEY_CLASSES_ROOT, subKey);
	return HRESULT_FROM_WIN32(result == ERROR_FILE_NOT_FOUND ? ERROR_SUCCESS : result);
}

} // namespace

GLOBAL(HRESULT) DllNotifyShell() {
	SHChangeNotify(SHCNE_ASSOCCHANGED, SHCNF_IDLIST, nullptr, nullptr);
	return S_OK;
}

GLOBAL(HRESULT) DllRegisterServer() {
	wchar_t modulePath[MAX_PATH];
	if (!GetModuleFileNameW(g_hInst, modulePath, MAX_PATH)) {
		return HRESULT_FROM_WIN32(GetLastError());
	}
	HRESULT hr;
	hr = ::SetHKCRRegistryKey(L"CLSID\\" VTF_THUMBNAILER_CLSID_STR, nullptr, L"Valve Texture Format Files (VTF Thumbnailer)");
	if (FAILED(hr)) {
		return hr;
	}
	hr = ::SetHKCRRegistryKey(L"CLSID\\" VTF_THUMBNAILER_CLSID_STR "\\InProcServer32", nullptr, modulePath);
	if (FAILED(hr)) {
		return hr;
	}
	hr = ::SetHKCRRegistryKey(L"CLSID\\" VTF_THUMBNAILER_CLSID_STR "\\InProcServer32", L"ThreadingModel", L"Apartment");
	if (FAILED(hr)) {
		return hr;
	}
	hr = ::SetHKCRRegistryKey(L".vtf\\ShellEx\\{e357fccd-a995-4576-b01f-234630154e96}", nullptr, VTF_THUMBNAILER_CLSID_STR);
	if (SUCCEEDED(hr)) {
		::DllNotifyShell();
	}
	return hr;
}

GLOBAL(HRESULT) DllUnregisterServer() {
	HRESULT hr;
	hr = ::DeleteHKCRRegistryKey(L"CLSID\\" VTF_THUMBNAILER_CLSID_STR);
	if (FAILED(hr)) {
		return hr;
	}
	return ::DeleteHKCRRegistryKey(L".vtf\\ShellEx\\{e357fccd-a995-4576-b01f-234630154e96}");
}
{{</code>}}

The `DllRegisterServer` and `DllUnregisterServer` functions are called in the installer after the files are copied into
place through `regsvr32.exe`. Passing a path to a DLL to this program will call the `DllRegisterServer` function in the
DLL, which in this case modifies the necessary registry values, lets the file browser know there's a new thumbnail
provider, and exits. Running the same command with the `/u` flag will call the `DllUnregisterServer` function in the DLL
instead.

As for the specific registry keys, each one resides under `HKEY_CLASSES_ROOT`. The `CLSID\<GUID>` value is the thumbnail
provider name. The value for `CLSID\<GUID>\InProcServer32` is the path to the DLL. The value for
`CLSID\<GUID>\InProcServer32\ThreadingModel` should always be `Apartment` according to the scant Windows docs on
thumbnail providers and every example of one I've found. Finally, the value for
`<file extension>\ShellEx\{e357fccd-a995-4576-b01f-234630154e96}` (yes, you need that exact GUID) should be your own
thumbnail provider's GUID. These are all Windows needs to find the DLL, load it correctly, and use it on the files you
want thumbnails for.

Oh right and here's the stupid `.def` file, because the Windows API is just the worst.

{{<code language="def" title="Windows Thumbnail Provider BS - Exports">}}
LIBRARY	"vtf-thumbnailer"
EXPORTS
	VTFThumbnailProvider_CreateInstance PRIVATE
	DllMain                             PRIVATE
	DllAddRef                           PRIVATE
	DllRelease                          PRIVATE
	DllCanUnloadNow                     PRIVATE
	DllGetClassObject                   PRIVATE
	DllNotifyShell                      PRIVATE
	DllRegisterServer                   PRIVATE
	DllUnregisterServer                 PRIVATE
{{</code>}}

---

One DLL is written when installing this application. It should start working immediately without rebooting.

- `C:\Program Files\vtf-thumbnailer\vtf-thumbnailer.dll`

## Wrap-Up

At this point I was tired of thumbnailers. I packaged the product and shipped a release, built from the code hosted on
[this GitHub repository](https://github.com/craftablescience/vtf-thumbnailer), and haven't looked back until now. I hope
this post helps and/or inspires you to make a thumbnailer of your own, it's really not that difficult if you have a
decent reference and quite rewarding in the end!
