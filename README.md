import { useEffect, useState } from "react";
import { useNavigate } from "react-router-dom";
import { supabase } from "@/integrations/supabase/client";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Avatar, AvatarFallback } from "@/components/ui/avatar";
import { LogOut, MessageCircle, Search, UserPlus } from "lucide-react";
import { useToast } from "@/hooks/use-toast";
import { formatDistanceToNow } from "date-fns";

interface Profile {
  id: string;
  display_name: string | null;
  phone_number: string;
  last_seen: string;
  is_online: boolean;
}

const Contacts = () => {
  const [contacts, setContacts] = useState<Profile[]>([]);
  const [currentUserId, setCurrentUserId] = useState<string>("");
  const [loading, setLoading] = useState(true);
  const [searchPhone, setSearchPhone] = useState("");
  const [searching, setSearching] = useState(false);
  const navigate = useNavigate();
  const { toast } = useToast();

  useEffect(() => {
    checkAuth();
    fetchContacts();
  }, []);

  const checkAuth = async () => {
    const { data: { user } } = await supabase.auth.getUser();
    if (!user) {
      navigate("/auth");
      return;
    }
    setCurrentUserId(user.id);
  };

  const fetchContacts = async () => {
    try {
      const { data, error } = await supabase
        .from("profiles")
        .select("*")
        .order("display_name");

      if (error) throw error;
      
      // Filter out current user
      const { data: { user } } = await supabase.auth.getUser();
      const filteredContacts = data?.filter((profile) => profile.id !== user?.id) || [];
      setContacts(filteredContacts);
    } catch (error: any) {
      toast({
        title: "Error",
        description: error.message,
        variant: "destructive",
      });
    } finally {
      setLoading(false);
    }
  };

  const handleSignOut = async () => {
    await supabase.auth.signOut();
    navigate("/auth");
  };

  const startChat = (contactId: string) => {
    navigate(`/chat/${contactId}`);
  };

  const searchUserByPhone = async () => {
    if (!searchPhone.trim()) {
      toast({
        title: "Error",
        description: "Please enter a phone number",
        variant: "destructive",
      });
      return;
    }

    // Validate phone number format (10-15 digits)
    const phoneRegex = /^[0-9]{10,15}$/;
    if (!phoneRegex.test(searchPhone.replace(/[\s\-()]/g, ""))) {
      toast({
        title: "Invalid Format",
        description: "Please enter a valid phone number (10-15 digits)",
        variant: "destructive",
      });
      return;
    }

    setSearching(true);
    try {
      // Search for user by phone number (stored as email format)
      const phoneEmail = `${searchPhone}@chatapp.com`;
      const { data, error } = await supabase
        .from("profiles")
        .select("*")
        .eq("phone_number", phoneEmail)
        .single();

      if (error || !data) {
        toast({
          title: "Not Found",
          description: "No user found with this phone number",
          variant: "destructive",
        });
        return;
      }

      // Navigate to chat with this user
      startChat(data.id);
    } catch (error: any) {
      toast({
        title: "Error",
        description: error.message,
        variant: "destructive",
      });
    } finally {
      setSearching(false);
      setSearchPhone("");
    }
  };

  const getInitials = (name: string | null) => {
    if (!name) return "U";
    return name
      .split(" ")
      .map((n) => n[0])
      .join("")
      .toUpperCase()
      .slice(0, 2);
  };

  if (loading) {
    return (
      <div className="flex min-h-screen items-center justify-center">
        <div className="text-muted-foreground">Loading contacts...</div>
      </div>
    );
  }

  return (
    <div className="flex flex-col h-screen bg-background">
      {/* Header */}
      <header className="border-b bg-card">
        <div className="flex items-center justify-between p-4">
          <div className="flex items-center gap-2">
            <MessageCircle className="h-6 w-6 text-primary" />
            <h1 className="text-xl font-semibold">Chats</h1>
          </div>
          <Button variant="ghost" size="icon" onClick={handleSignOut}>
            <LogOut className="h-5 w-5" />
          </Button>
        </div>
        
        {/* Search Section */}
        <div className="p-4 border-t bg-background">
          <div className="flex gap-2">
            <div className="relative flex-1">
              <Search className="absolute left-3 top-1/2 -translate-y-1/2 h-4 w-4 text-muted-foreground" />
              <Input
                type="text"
                placeholder="Enter phone number to start chat"
                value={searchPhone}
                onChange={(e) => setSearchPhone(e.target.value)}
                onKeyDown={(e) => e.key === "Enter" && searchUserByPhone()}
                className="pl-10"
              />
            </div>
            <Button 
              onClick={searchUserByPhone} 
              disabled={searching}
              size="icon"
            >
              <UserPlus className="h-5 w-5" />
            </Button>
          </div>
        </div>
      </header>

      {/* Contacts List */}
      <div className="flex-1 overflow-y-auto">
        {contacts.length === 0 ? (
          <div className="flex flex-col items-center justify-center h-full text-center p-8">
            <MessageCircle className="h-16 w-16 text-muted-foreground mb-4" />
            <p className="text-muted-foreground mb-2">
              No chats yet
            </p>
            <p className="text-sm text-muted-foreground">
              Search for a phone number above to start chatting
            </p>
          </div>
        ) : (
          <div className="divide-y">
            {contacts.map((contact) => (
              <button
                key={contact.id}
                onClick={() => startChat(contact.id)}
                className="w-full p-4 flex items-center gap-3 hover:bg-secondary/50 transition-colors"
              >
                <Avatar className="h-12 w-12">
                  <AvatarFallback className="bg-primary/10 text-primary">
                    {getInitials(contact.display_name)}
                  </AvatarFallback>
                </Avatar>
                <div className="flex-1 text-left">
                  <div className="flex items-center gap-2">
                    <h3 className="font-semibold">
                      {contact.display_name || "User"}
                    </h3>
                    {contact.is_online && (
                      <span className="h-2 w-2 rounded-full bg-[hsl(var(--online-status))]" />
                    )}
                  </div>
                  <p className="text-sm text-muted-foreground">
                    {contact.is_online
                      ? "Online"
                      : `Last seen ${formatDistanceToNow(new Date(contact.last_seen), { addSuffix: true })}`}
                  </p>
                </div>
              </button>
            ))}
          </div>
        )}
      </div>
    </div>
  );
};

export default Contacts;
