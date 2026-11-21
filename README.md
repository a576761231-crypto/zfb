# zfbTARGET := iphone:clang:latest:15.0
ARCHS := arm64

include $(THEOS)/makefiles/common.mk

LIBRARY_NAME = ZfbTabTrim
ZfbTabTrim_FILES = Tweak.m
ZfbTabTrim_CFLAGS = -fobjc-arc
ZfbTabTrim_INSTALL_PATH = @rpath

include $(THEOS_MAKE_PATH)/library.mk
name: build-dylib
on:
  workflow_dispatch:
  push:
jobs:
  build:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - run: brew install ldid make
      - name: Install Theos and SDK
        run: |
          git clone --recursive https://github.com/theos/theos.git "$HOME/theos"
          echo "THEOS=$HOME/theos" >> "$GITHUB_ENV"
          curl -sL https://github.com/theos/sdks/archive/master.tar.gz | tar -xz -C /tmp
          mkdir -p "$HOME/theos/sdks"
          cp -R /tmp/sdks-master/iPhoneOS16.5.sdk "$HOME/theos/sdks/"
      - run: gmake
      - name: Collect dylib
        run: |
          mkdir -p out
          cp "$(find .theos/obj -name '*.dylib' | head -1)" out/ZfbTabTrim.dylib
          otool -L out/ZfbTabTrim.dylib
      - uses: actions/upload-artifact@v4
        with:
          name: ZfbTabTrim-dylib
          path: out/ZfbTabTrim.dylib
          #import <UIKit/UIKit.h>
#import <objc/runtime.h>

static NSArray *TitleList(void) { return @[@"理财", @"财富", @"Wealth", @"Fortune"]; }
static NSArray *ClassList(void) { return @[@"FHMainViewController", @"HFNTabMainController"]; }

static BOOL IsWealth(UIViewController *vc) {
    if (!vc) return NO;
    NSString *cls = NSStringFromClass([vc class]);
    NSString *t1 = vc.tabBarItem.title ?: @"";
    NSString *t2 = vc.title ?: @"";
    NSLog(@"[ZfbTabTrim] vc=%@ tabItemTitle=%@ title=%@", cls, t1, t2);
    if ([ClassList() containsObject:cls]) return YES;
    if ([TitleList() containsObject:t1] || [TitleList() containsObject:t2]) return YES;
    if ([vc isKindOfClass:[UINavigationController class]]) {
        for (UIViewController *c in ((UINavigationController *)vc).viewControllers)
            if (IsWealth(c)) return YES;
    }
    return NO;
}

static NSArray *Filter(NSArray *vcs) {
    if (![vcs isKindOfClass:[NSArray class]] || vcs.count < 3) return vcs;
    NSMutableArray *out = [NSMutableArray arrayWithCapacity:vcs.count];
    for (UIViewController *vc in vcs) if (!IsWealth(vc)) [out addObject:vc];
    return out.count == vcs.count ? vcs : out;
}

static void (*orig_set)(id, SEL, NSArray *);
static void new_set(id self, SEL _cmd, NSArray *vcs) { orig_set(self, _cmd, Filter(vcs)); }

static void (*orig_set_anim)(id, SEL, NSArray *, BOOL);
static void new_set_anim(id self, SEL _cmd, NSArray *vcs, BOOL a) { orig_set_anim(self, _cmd, Filter(vcs), a); }

static IMP Hook(Class cls, SEL sel, IMP newImp) {
    Method m = class_getInstanceMethod(cls, sel);
    if (!m) return NULL;
    IMP old = method_getImplementation(m);
    class_replaceMethod(cls, sel, newImp, method_getTypeEncoding(m));
    return old;
}

static void Install(void) {
    Class cls = NSClassFromString(@"ALPLauncherController");
    if (!cls) {
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.5 * NSEC_PER_SEC)),
                       dispatch_get_main_queue(), ^{ Install(); });
        return;
    }
    orig_set      = (void *)Hook(cls, @selector(setViewControllers:), (IMP)new_set);
    orig_set_anim = (void *)Hook(cls, @selector(setViewControllers:animated:), (IMP)new_set_anim);
    NSLog(@"[ZfbTabTrim] installed %d %d", orig_set != NULL, orig_set_anim != NULL);
}

__attribute__((constructor)) static void ZfbTabTrimInit(void) { Install(); }
