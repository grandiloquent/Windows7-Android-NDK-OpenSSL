# Windows7-Android-NDK-OpenSSL

1. 创建工具链

		ANDROID_NDK/build/tools/make_standalone_toolchain.py --arch arm --api 19 --install-dir ANDROID_NDK/toolchains/android-19-arm

2. 创建 Shell Script

		#!/bin/sh
	
		export ANDROID_NDK=C:/Users/psycho/AppData/Local/Android/Sdk/ndk-bundle/toolchains/android-19-arm/
		#export CROSS_SYSROOT=C:/Users/psycho/AppData/Local/Android/Sdk/ndk-bundle/sysroot/
		export PERL=C:/msys64/usr/bin/perl
		export PATH=C:/Users/psycho/AppData/Local/Android/Sdk/ndk-bundle/toolchains/android-19-arm/bin/
		export CC=clang
	
		_OUTPUT_DIR=C:/openssl-output
		_ANDROID_API=19
		_ANDROID_ARCH=android-arm
	
		make clean
		rm -rf $_OUTPUT_DIR/android-$_ANDROID_API/$_ANDROID_ARCH
		mkdir -p $_OUTPUT_DIR/android-$_ANDROID_API/$_ANDROID_ARCH
	
		./Configure android-arm -D__ANDROID_API__=19 --openssldir=$_OUTPUT_DIR/android-$_ANDROID_API/$_ANDROID_ARCH --prefix=$_OUTPUT_DIR/android-$_ANDROID_API/$_ANDROID_ARCH	
	
	
		

3. 修改  `openssl-1.1.1c/Configure` 

		#! C:/msys64/usr/bin/perl

4. 修改  `openssl-1.1.1c/Configurations/15-android.conf` 
	
		#### Android...
		#
		# See NOTES.ANDROID for details, and don't miss platform-specific
		# comments below...
	
		{
		    use File::Spec::Functions;
	
		    my $android_ndk = {};
		    my %triplet = (
		        arm    => "arm-linux-androideabi",
		        arm64  => "aarch64-linux-android",
		        mips   => "mipsel-linux-android",
		        mips64 => "mips64el-linux-android",
		        x86    => "i686-linux-android",
		        x86_64 => "x86_64-linux-android",
		    );
	
		    sub android_ndk {
		        unless (%$android_ndk) {
		            if ($now_printing =~ m|^android|) {
		                return $android_ndk = { bn_ops => "BN_AUTO" };
		            }
	
		            my $ndk_var;
		            my $ndk;
		            foreach (qw(ANDROID_NDK_HOME ANDROID_NDK)) {
		                $ndk_var = $_;
		                $ndk = $ENV{$ndk_var};
		                last if defined $ndk;
		            }
		            die "\$ANDROID_NDK_HOME is not defined"  if (!$ndk);
		            if (!-d "$ndk/platforms" && !-f "$ndk/AndroidVersion.txt") {
		                # $ndk/platforms is traditional "all-inclusive" NDK, while
		                # $ndk/AndroidVersion.txt is so-called standalone toolchain
		                # tailored for specific target down to API level.
		                die "\$ANDROID_NDK_HOME=$ndk is invalid";
		            }
		            $ndk = canonpath($ndk);
	
		            my $ndkver = undef;
	
		            if (open my $fh, "<$ndk/source.properties") {
		                local $_;
		                while(<$fh>) {
		                    if (m|Pkg\.Revision\s*=\s*([0-9]+)|) {
		                        $ndkver = $1;
		                        last;
		                    }
		                }
		                close $fh;
		            }
	
		            my ($sysroot, $api, $arch);
	
		            $config{target} =~ m|[^-]+-([^-]+)$|;	# split on dash
		            $arch = $1;
	
		            if ($sysroot = $ENV{CROSS_SYSROOT}) {
		                $sysroot =~ m|/android-([0-9]+)/arch-(\w+)/?$|;
		                ($api, $arch) = ($1, $2);
		            } elsif (-f "$ndk/AndroidVersion.txt") {
		                $sysroot = "$ndk/sysroot";
		            } else {
		                $api = "*";
	
		                # see if user passed -D__ANDROID_API__=N
		                foreach (@{$useradd{CPPDEFINES}}, @{$user{CPPFLAGS}}) {
		                    if (m|__ANDROID_API__=([0-9]+)|) {
		                        $api = $1;
		                        last;
		                    }
		                }
	
		                # list available platforms (numerically)
		                my @platforms = sort { $a =~ m/-([0-9]+)$/; my $aa = $1;
		                                       $b =~ m/-([0-9]+)$/; $aa <=> $1;
		                                     } glob("$ndk/platforms/android-$api");
		                die "no $ndk/platforms/android-$api" if ($#platforms < 0);
	
		                $sysroot = "@platforms[$#platforms]/arch-$arch";
		                $sysroot =~ m|/android-([0-9]+)/arch-$arch|;
		                $api = $1;
		            }
		            die "no sysroot=$sysroot"   if (!-d $sysroot);
	
		            my $triarch = $triplet{$arch};
		            my $cflags;
		            my $cppflags;
					printf "%s %s\n",$arch,$sysroot;
	
		            # see if there is NDK clang on $PATH, "universal" or "standalone"
		            if (which("clang") =~ m|^$ndk/.*/prebuilt/([^/]+)/|) {
		                my $host=$1;
		                # harmonize with gcc default
		                my $arm = $ndkver > 16 ? "armv7a" : "armv5te";
		                (my $tridefault = $triarch) =~ s/^arm-/$arm-/;
		                (my $tritools   = $triarch) =~ s/(?:x|i6)86(_64)?-.*/x86$1/;
		                $cflags .= " -target $tridefault "
		                        .  "-gcc-toolchain \$($ndk_var)/toolchains"
		                        .  "/$tritools-4.9/prebuilt/$host";
		                $user{CC} = "clang" if ($user{CC} !~ m|clang|);
		                $user{CROSS_COMPILE} = undef;
		                if (which("llvm-ar") =~ m|^$ndk/.*/prebuilt/([^/]+)/|) {
		                    $user{AR} = "llvm-ar";
		                    $user{ARFLAGS} = [ "rs" ];
		                    $user{RANLIB} = ":";
		                }
		            } elsif (-f "$ndk/AndroidVersion.txt") {    #"standalone toolchain"
		                my $cc = $user{CC} // "clang";
		                # One can probably argue that both clang and gcc should be
		                # probed, but support for "standalone toolchain" was added
		                # *after* announcement that gcc is being phased out, so
		                # favouring clang is considered adequate. Those who insist
		                # have option to enforce test for gcc with CC=gcc.
	
		                #if (which("$triarch-$cc") !~ m|^$ndk|) {
		                #    die "no NDK $triarch-$cc on \$PATH";
		                # }
		                $user{CC} = $cc;
		                $user{CROSS_COMPILE} = "$triarch-";
		            } elsif ($user{CC} eq "clang") {
		                die "no NDK clang on \$PATH";
		            } else {
		                if (which("$triarch-gcc") !~ m|^$ndk/.*/prebuilt/([^/]+)/|) {
		                    die "no NDK $triarch-gcc on \$PATH";
		                }
		                $cflags .= " -mandroid";
		                $user{CROSS_COMPILE} = "$triarch-";
		            }
	
					printf "%s,%s\n",$sysroot,$ndk_var;
	
		            if (!-d "$sysroot/usr/include") {
						printf "%s\n","$sysroot/usr/include";
		                my $incroot = "$ndk/sysroot/usr/include";
		                die "no $incroot"          if (!-d $incroot);
		                die "no $incroot/$triarch" if (!-d "$incroot/$triarch");
		                $incroot =~ s|^$ndk/||;
		                $cppflags  = "-D__ANDROID_API__=$api";
		                $cppflags .= " -isystem \$($ndk_var)/$incroot/$triarch";
		                $cppflags .= " -isystem \$($ndk_var)/$incroot";
		            }
	
		            $sysroot =~ s|^$ndk/||;
		            $android_ndk = {
		                cflags   => "$cflags --sysroot=C:/Users/psycho/AppData/Local/Android/Sdk/ndk-bundle/toolchains/android-19-arm/$sysroot",
		                cppflags => $cppflags,
		                bn_ops   => $arch =~ m/64$/ ? "SIXTY_FOUR_BIT_LONG"
		                                            : "BN_LLONG",
		            };
		        }
	
		        return $android_ndk;
		    }
		}
	
		my %targets = (
		    "android" => {
		        inherit_from     => [ "linux-generic32" ],
		        template         => 1,
		        ################################################################
		        # Special note about -pie. The underlying reason is that
		        # Lollipop refuses to run non-PIE. But what about older systems
		        # and NDKs? -fPIC was never problem, so the only concern is -pie.
		        # Older toolchains, e.g. r4, appear to handle it and binaries
		        # turn out mostly functional. "Mostly" means that oldest
		        # Androids, such as Froyo, fail to handle executable, but newer
		        # systems are perfectly capable of executing binaries targeting
		        # Froyo. Keep in mind that in the nutshell Android builds are
		        # about JNI, i.e. shared libraries, not applications.
		        cflags           => add(sub { android_ndk()->{cflags} }),
		        cppflags         => add(sub { android_ndk()->{cppflags} }),
		        cxxflags         => add(sub { android_ndk()->{cflags} }),
		        bn_ops           => sub { android_ndk()->{bn_ops} },
		        bin_cflags       => "-pie",
		        enable           => [ ],
		    },
		    "android-arm" => {
		        ################################################################
		        # Contemporary Android applications can provide multiple JNI
		        # providers in .apk, targeting multiple architectures. Among
		        # them there is "place" for two ARM flavours: generic eabi and
		        # armv7-a/hard-float. However, it should be noted that OpenSSL's
		        # ability to engage NEON is not constrained by ABI choice, nor
		        # is your ability to call OpenSSL from your application code
		        # compiled with floating-point ABI other than default 'soft'.
		        # (Latter thanks to __attribute__((pcs("aapcs"))) declaration.)
		        # This means that choice of ARM libraries you provide in .apk
		        # is driven by application needs. For example if application
		        # itself benefits from NEON or is floating-point intensive, then
		        # it might be appropriate to provide both libraries. Otherwise
		        # just generic eabi would do. But in latter case it would be
		        # appropriate to
		        #
		        #   ./Configure android-arm -D__ARM_MAX_ARCH__=8
		        #
		        # in order to build "universal" binary and allow OpenSSL take
		        # advantage of NEON when it's available.
		        #
		        # Keep in mind that (just like with linux-armv4) we rely on
		        # compiler defaults, which is not necessarily what you had
		        # in mind, in which case you would have to pass additional
		        # -march and/or -mfloat-abi flags. NDK defaults to armv5te.
		        # Newer NDK versions reportedly require additional -latomic.
		        #
		        inherit_from     => [ "android", asm("armv4_asm") ],
		        bn_ops           => add("RC4_CHAR"),
		    },
		    "android-arm64" => {
		        inherit_from     => [ "android", asm("aarch64_asm") ],
		        bn_ops           => add("RC4_CHAR"),
		        perlasm_scheme   => "linux64",
		    },
	
		    "android-mips" => {
		        inherit_from     => [ "android", asm("mips32_asm") ],
		        bn_ops           => add("RC4_CHAR"),
		        perlasm_scheme   => "o32",
		    },
		    "android-mips64" => {
		        ################################################################
		        # You are more than likely have to specify target processor
		        # on ./Configure command line. Trouble is that toolchain's
		        # default is MIPS64r6 (at least in r10d), but there are no
		        # such processors around (or they are too rare to spot one).
		        # Actual problem is that MIPS64r6 is binary incompatible
		        # with previous MIPS ISA versions, in sense that unlike
		        # prior versions original MIPS binary code will fail.
		        #
		        inherit_from     => [ "android", asm("mips64_asm") ],
		        bn_ops           => add("RC4_CHAR"),
		        perlasm_scheme   => "64",
		    },
	
		    "android-x86" => {
		        inherit_from     => [ "android", asm("x86_asm") ],
		        CFLAGS           => add(picker(release => "-fomit-frame-pointer")),
		        bn_ops           => add("RC4_INT"),
		        perlasm_scheme   => "android",
		    },
		    "android-x86_64" => {
		        inherit_from     => [ "android", asm("x86_64_asm") ],
		        bn_ops           => add("RC4_INT"),
		        perlasm_scheme   => "elf",
		    },
	
		    ####################################################################
		    # Backward compatible targets, (might) requre $CROSS_SYSROOT
		    #
		    "android-armeabi" => {
		        inherit_from     => [ "android-arm" ],
		    },
		    "android64" => {
		        inherit_from     => [ "android" ],
		    },
		    "android64-aarch64" => {
		        inherit_from     => [ "android-arm64" ],
		    },
		    "android64-x86_64" => {
		        inherit_from     => [ "android-x86_64" ],
		    },
		    "android64-mips64" => {
		        inherit_from     => [ "android-mips64" ],
		    },
		);	

5. 设置环境变量 

		set path=%path%;C:\msys64\usr\bin
		set path=%path%;C:/Users/psycho/AppData/Local/Android/Sdk/ndk-bundle/toolchains/android-19-arm/bin
		mkdir c:\openssl-output

6. 编译

		cd C:\library\openssl-1.1.1c
		sh build_ssl.sh
		make
		make install
