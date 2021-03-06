# HotPatch
A single-header C++ library for Windows to create hotpatchable images and apply hotpatches with 5 methods

Article at CodeProject: https://www.codeproject.com/Articles/1043089/HotPatching-Deep-Inside

## Executable to be hot-patched preparation

1. Build the executable in ***release*** mode from solution
2. The solution is automatically configured to run the executable with parameter /postbuildpatch which updates itself with the patch information. It uses BeginUpdateResource and EndUpdateResource.
Note that these are frequently stopped by antivirus. You can also use the included pig.exe which is a standalone app to read a file and its pdf, and put the hotpatch data inside it.

## Method 1: Using a DLL to patch an Executable 

1. Load the DLL from the executable and call an exported function (say, Patch()).
2. Call hp.ApplyPatchFor() for each function you want to be patched:


```C++
hr = hp.ApplyPatchFor(hM, L"FOO::PatchableFunction1", PatchableFunction1, &xPatch);
```

## Method 2: Using the same executable as a COM server

1. Call hp.PrepareExecutableForCOMPatching();
2. If embedding, start the COM server, specifying the patches and installing a message loop:

```C++

void EmbeddingStart()
{
	hp.StartCOMServer(GUID_TEST, [](vector<HOTPATCH::NAMEANDPOINTER>& w) -> HRESULT
	{
		w.clear();
		HOTPATCH::NAMEANDPOINTER nap;
		nap.n = L"PatchableFunction1";
		nap.f = [](size_t*) -> size_t
		{
			MessageBox(0, L"Patch from COM Patcher", L"Patched", MB_ICONINFORMATION);
			return 0;
		};
		w.push_back(nap);
		return S_OK;
	},

	[]()
	{
		// We are closing...
		PostThreadMessage(mtid, WM_QUIT, 0, 0);
	}
	);
}

// main
if (argc == 2 && (_wcsicmp(wargv[1], L"-embedding") == 0 || _wcsicmp(wargv[1], L"/embedding") == 0))
{
	mtid = GetCurrentThreadId();
	EmbeddingStart();
	MSG msg;
	while (GetMessage(&msg, 0, 0, 0))
	{
		TranslateMessage(&msg);
		DispatchMessage(&msg);
	}
	return 0;
}
	
```

If main, start the patching process

```C++
hp.StartCOMPatching();
vector<wstring> pns;
hp.AckGetPatchNames(pns);
for (auto& aa : pns)
{
	hp.ApplyCOMPatchFor(xPatch, GetModuleHandle(0), aa.c_str());
}
...
// End app
hp.FinishCOMPatching();
```

## Method 3: Using the same executable with shared memory

1. Call hp.PrepareExecutableForUSMPatching();
2. If patching mode, start the USM server, specifying the patches and installing a message loop:

```C++

void USMStart()
{
	hp.StartUSMServer(GUID_TEST, [](vector<HOTPATCH::NAMEANDPOINTER>& w) -> HRESULT
	{
		HOTPATCH::NAMEANDPOINTER nap;
		nap.n = L"PatchableFunction1";

		TCHAR cidx[1000] = { 0 };
		StringFromGUID2(GUID_TEST, cidx, 1000);
		swprintf_s(cidx + wcslen(cidx), 1000 - wcslen(cidx), L"-%u", 0);
		nap.mu = (mutual*)new mutual(cidx, [&](unsigned long long)
		{
			MessageBox(0, L"Patch from USM Patcher", L"Patched", MB_ICONINFORMATION);
			return;
		}, 0, true);
		w.push_back(nap);

		return S_OK;
	},

		[]()
	{
		// We are closing...
		PostThreadMessage(mtid, WM_QUIT, 0, 0);
	}
	);

}



// main
if (argc == 2 && (_wcsicmp(wargv[1], L"-usm") == 0 || _wcsicmp(wargv[1], L"/usm") == 0))
{
	mtid = GetCurrentThreadId();
	USMStart();
	MSG msg;
	while (GetMessage(&msg, 0, 0, 0))
	{
		TranslateMessage(&msg);
		DispatchMessage(&msg);
	}
	return 0;
}

	
```

If main, start the patching process

```C++
// a has path name to patcher
wcscat_s(a, 1000, L" /usm");

// Run it with /USM
STARTUPINFO sInfo = { 0 };
sInfo.cb = sizeof(sInfo);
PROCESS_INFORMATION pi = { 0 };
if (!CreateProcess(0, a, 0, 0, 0, 0, 0, 0, &sInfo, &pi))
	return 0;
CloseHandle(pi.hThread);
CloseHandle(pi.hProcess);

// Get the names
vector<wstring> pns;
hp.AckGetPatchNames(pns);
for (size_t i = 0; i < pns.size(); i++)
{
	auto& aa = pns[i];
	hp.ApplyUSMPatchFor(xPatch, GetModuleHandle(0), aa.c_str(), i);
}
...

// End app
hp.FinishCOMPatching();
```

## Method 4 : Self-EXE

This method is based on my article at https://www.codeproject.com/Articles/1045674/Load-EXE-as-DLL-Mission-Possible , which describes how to load an EXE as a DLL.
However this is a highly hackish method and should not be used.

## Method 5 : Self-DLL

The final method requires your app to be build itself as DLL. Therefore, the post-build event is now  *rundll32 .\test.dll,PostBuildPatch*, the debugger run command is now *rundll32 .\test.dll,dmain*
and you can have the program and the patcher inside the same DLL.





