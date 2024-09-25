# Creating Custom Controls

An introduction to creating custom controls using MFC



![Sample Image - CustomControl.jpg](https://raw.githubusercontent.com/ChrisMaunder/customcontrol/master/docs/assets/customcontrol.jpg)

## Introduction

In a [previous article](https://github.com/ChrisMaunder/subclassdemo)
I demonstrated subclassing a windows common control in order to modify its behaviour or
extend its functionality. Sometimes you can only push the windows common controls so far.
An example I came across was the common issue of needing a grid control to display and
edit tabular data. I subclassed a `CListCtrl`       
   and extended it to allow subitem editing, multiline cells, sort-on-click headers and 
a myriad of other features. However, deep down it was still a list control and there came
a point where I seemed to be writing more code to *stop* the control performing actions than I 
was to actually *make*  it do something. 

I needed to start from scratch, working from a base class that provided only the 
functionality I needed without any of the features (or liabilities) that I didn't
need. Enter the custom control. 

## Creating a Custom Control class

Writing a custom control is very similar to subclassing a windows common control.
You derive a new class from an existing class and override the functionality of the
base class in order to make it do what you want. 

In this case we'll be deriving a class from `CWnd`, since this class provides the minimum 
functionality we need, without too much overhead. 

The first step in creating a custom control is to derive your class from your chosen base
class (`CWnd`). In this example we'll create a custom control for displaying
bitmaps, and we'll call this class `CBitmapViewer`. Obviously there is already
the `CStatic` class that already displays bitmaps, but the example is only meant 
to demonstrate the possibilities available to the adventurous programmer.

![](https://raw.githubusercontent.com/ChrisMaunder/customcontrol/master/docs/assets/customcontrol3.gif) 

To your class you should add handlers for the `WM_PAINT` and `WM_ERASEBKGND`
messages. I've also added an override for `PreSubclassWindow` in case you wish to
perform any initialisation that requires the window to have been created. See my
[previous article](http://www.codeproject.com/miscctrl/subclassdemo.asp) for a discussion
of  `PreSubclassWindow`.

![](https://raw.githubusercontent.com/ChrisMaunder/customcontrol/master/docs/assets/customcontrol4.gif) 

The aim of this control is to display bitmaps, so we'll a method to set the bitmap and call it
`SetBitmap`. We're not only talented, us programmers, but extremely imaginative as well.

The internal code for the control is unimportant to this discussion but is included for completeness.

Add a member variable of type `CBitmap` to the class, as well as the `SetBitmap`
prototype:

```cpp
class CBitmapViewer : public CWnd
{
// Construction
public:
    CBitmapViewer();

// Attributes
public:
    BOOL SetBitmap(UINT nIDResource);

    ...
    
protected:
    CBitmap m_Bitmap;
};
```

In your `CBitmapViewer` implementation file add the following code for your `SetBitmap`
method, and your `WM_PAINT` and `WM_ERASEBKGND` message handlers:

```cpp
void CBitmapViewer::OnPaint() 
{
    // Draw the bitmap - if we have one.
    if (m_Bitmap.GetSafeHandle() != NULL)
    {
        CPaintDC dc(this); // device context for painting

        // Create memory DC
        CDC MemDC;
        if (!MemDC.CreateCompatibleDC(&dc))
            return;

        // Get Size of Display area
        CRect rect;
        GetClientRect(rect);

        // Get size of bitmap
        BITMAP bm;
        m_Bitmap.GetBitmap(&bm);
        
        // Draw the bitmap
        CBitmap* pOldBitmap = (CBitmap*) MemDC.SelectObject(&m_Bitmap);
        dc.StretchBlt(0, 0, rect.Width(), rect.Height(), 
                      &MemDC, 
                      0, 0, bm.bmWidth, bm.bmHeight, 
                      SRCCOPY);
        MemDC.SelectObject(pOldBitmap);      
    }
    
    // Do not call CWnd::OnPaint() for painting messages
}

BOOL CBitmapViewer::OnEraseBkgnd(CDC* pDC) 
{
    // If we have an image then don't perform any erasing, since the OnPaint
    // function will simply draw over the background
    if (m_Bitmap.GetSafeHandle() != NULL)
        return TRUE;
    
    // Obviously we don't have a bitmap - let the base class deal with it.
    return CWnd::OnEraseBkgnd(pDC);
}

BOOL CBitmapViewer::SetBitmap(UINT nIDResource)
{
    return m_Bitmap.LoadBitmap(nIDResource);
}
```

## Making the class a Custom Control

So far we have a class that allows us to load and display a bitmap - but as yet we have no
way of actually *using* this class. We have two choices in creating the control - either
dynamically by calling `Create` or via a dialog template created using the Visual
Studio resource editor.

Since our class is derived from `CWnd` we can use `CWnd::Create` to
create the control dynamically. For instance, in your dialog's `OnInitDialog` you
could have the following code:

```cpp
// CBitmapViewer m_Viewer; - declared in dialog class header

m_Viewer.Create(NULL, _T(""), WS_VISIBLE, CRect(0,0,100,100), this, 1);
m_Viewer.SetBitmap(IDB_BITMAP1);
```

where *m\_Viewer* is an object of type `CBitmapViewer` that is declared
in your dialogs header, and `IDB_BITMAP1` is the ID of a bitmap resource. The
control will be created and the bitmap will display. 

However, what if we wished to place the control in a dialog template using the Visual
Studio resource editor? For this we need to register a Windows Class name using the
`AfxRegisterClass` function. Registering a class allows us to specify the background
color, the cursor, and the style. See `AfxRegisterWndClass` in the docs for more
information. 

For this example we'll register a simple class and call it "MFCBitmapViewerCtrl". We only
need to register the control once, and a neat place to do this is in the constructor of the
class we are writing

```cpp
#define BITMAPVIEWER_CLASSNAME    _T("MFCBitmapViewerCtrl")  // Window class name

CBitmapViewer::CBitmapViewer()
{
    RegisterWindowClass();
}

BOOL CBitmapViewer::RegisterWindowClass()
{
    WNDCLASS wndcls;
    HINSTANCE hInst = AfxGetInstanceHandle();

    if (!(::GetClassInfo(hInst, BITMAPVIEWER_CLASSNAME, &wndcls)))
    {
        // otherwise we need to register a new class
        wndcls.style            = CS_DBLCLKS | CS_HREDRAW | CS_VREDRAW;
        wndcls.lpfnWndProc      = ::DefWindowProc;
        wndcls.cbClsExtra       = wndcls.cbWndExtra = 0;
        wndcls.hInstance        = hInst;
        wndcls.hIcon            = NULL;
        wndcls.hCursor          = AfxGetApp()->LoadStandardCursor(IDC_ARROW);
        wndcls.hbrBackground    = (HBRUSH) (COLOR_3DFACE + 1);
        wndcls.lpszMenuName     = NULL;
        wndcls.lpszClassName    = BITMAPVIEWER_CLASSNAME;

        if (!AfxRegisterClass(&wndcls))
        {
            AfxThrowResourceException();
            return FALSE;
        }
    }

    return TRUE;
}
```

In our example of creating the control dynamically, we should now change the creation call to

```cpp
m_Viewer.Create(_T("MFCBitmapViewerCtrl"), _T(""), WS_VISIBLE, CRect(0,0,100,100), this, 1);
```

This will ensure the correct window styles, cursors and colors are used in the control. It's probably
worthwhile writing a new `Create` function for your custom control so that users don't
have to remember the window class name. For example:

```cpp
BOOL CBitmapViewer::Create(CWnd* pParentWnd, const RECT& rect, UINT nID, DWORD dwStyle /*=WS_VISIBLE*/)
{
    return CWnd::Create(BITMAPVIEWER_CLASSNAME, _T(""), dwStyle, rect, pParentWnd, nID);
}
```

To use the custom control in a dialog resource, simply create a custom control on the dialog
resource as you would any other control

![](https://raw.githubusercontent.com/ChrisMaunder/customcontrol/master/docs/assets/customcontrol1.gif)

and then in the control's properties, specify the class name as "MFCBitmapViewerCtrl"

![](https://raw.githubusercontent.com/ChrisMaunder/customcontrol/master/docs/assets/customcontrol2.gif)

The final step is to link up a member variable with the control. Simply declare an object
of type  `CBitmapViewer` in your dialog class (say, *m\_Viewer*) and in your
dialog's `DoDataExchange` add the following

```cpp
void CCustomControlDemoDlg::DoDataExchange(CDataExchange* pDX)
{
    CDialog::DoDataExchange(pDX);
    //{{AFX_DATA_MAP(CCustomControlDemoDlg)
    DDX_Control(pDX, IDC_CUSTOM1, m_Viewer);
    //}}AFX_DATA_MAP
}
```

`DDX_Control` links the member variable *m\_Viewer* with the control with
ID `IDC_CUSTOM1` by calling `SubclassWindow`. Creating a custom
control in your dialog resource with the class name "MFCBitmapViewerCtrl" creates a window
that behaves as your `CBitmapViewer::RegisterWindowClass` has specified, and
then the `DDX_Control` call links your `CBitmapViewer` object with
this pre-prepared window.

Compile your project, run the application, and be amazed. You've just created a custom control.
